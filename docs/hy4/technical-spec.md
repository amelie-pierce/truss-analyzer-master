# Truss Analyzer — Technical specification

**Status:** Code-derived baseline and proposed architecture for the UI rework.  
**Companion:** [UI rework specification](ui-rework-spec.md).  
**Reading guide:** Sections 1–6 describe current behavior. Sections 7–8 define the proposed application-layer changes and verification plan. Existing heuristics are documented, not endorsed as engineering rules.

## 1. System boundary

Truss Analyzer is a React/TypeScript single-page app served by Vite. It reads IFC and MiTek TRE text in the browser, builds girder-to-carried-truss relationships, displays drawing and load data, queries Simpson Strong-Tie (SST) for candidate hangers, and creates an Excel hanger schedule.

- **Local to the browser:** IFC/TRE and inventory file reads, connection analysis, diagrams, candidate ranking, and workbook generation. There is no application backend or job database in this repo.
- **External integration:** The browser sends a derived connection payload to SST for a lookup, not the original IFC, TRE, or inventory XML files. The app uses a bearer token stored in browser `localStorage`; a token being saved does not verify access. Requests depend on network and cross-origin access.
- **Persistence:** Job data, selection, inventory, and hanger results are held in React memory. Reloading starts a new session with the bundled sample; the token can persist. Excel is a download, not a resumable job file.
- **Engineering boundary:** The parser extracts loads and applies matching and fallback rules. It does not independently solve the truss or approve a hanger for construction.

## 2. Layers and ownership

| Layer | Current code | Responsibility |
| --- | --- | --- |
| Entry and shell | `src/main.tsx`, `src/App.tsx` | Mount React; load the bundled sample; own active girder groups, selected girder, active tab, logs, shared inventory, and shared hanger results. |
| Presentation and orchestration | `src/components/FileUpload.tsx`, `GirderList.tsx`, `GirderDetails.tsx`, `SSTWorkspace.tsx`, `JobSummaryTable.tsx` | Read user input, manage local interaction state, show diagrams and results, trigger single/batch lookups, rank summary candidates, and export Excel. Orchestration and presentation are currently mixed. |
| Domain model | `src/types.ts` | `TreData` (design/profile), `TrussInstance` (occurrence, possibly synthetic), `CarriedTruss` (a particular connection), `GirderGroup` (girder and its connections), and diagnostics. |
| Analysis | `src/lib/parser.ts` | Parse file content, associate profiles with instances, infer connections, and enrich each connection with bearing and reaction data. Does not call SST or export files. |
| SST mapping and contract | `src/lib/sst-mapper.ts`, `src/lib/sst-types.ts` | Translate a `GirderGroup` and `CarriedTruss` into SST units, member properties, load durations, and API enum values. |
| Adapters and output | `src/lib/sst-api.ts`, `src/lib/inventory.ts`, workbook code in `JobSummaryTable.tsx` | Authenticate and normalize SST responses; read local XML stock flags; build the `.xlsx` artifact. |

`ExportTab.tsx`, `HangerPanel.tsx`, and `LogPanel.tsx` exist but are not mounted in the current main flow. The active summary implements its own batch loop rather than using `submitBatchToSST`.

## 3. Data model and lifecycle

```text
IFC file(s) + TRE file(s)
  → TRE profiles by label + IFC instances
  → GirderGroup[] (connection offsets, spacing, reactions, bearing side)
  → Analyzer table and diagrams
  → SSTPayload for one connection → SST → candidate hanger list
  → candidate list + optional inventory → selected hanger per connection
  → job totals, model quantities, and Excel workbook
```

| Record | Meaning and identity |
| --- | --- |
| `TreData` | Design profile keyed by label. Includes available dimensions, chord/web data, cutting info, heel/stub values, slopes, loads, and hanger positions. The last TRE parsed with the same key overwrites the earlier profile. |
| `TrussInstance` | A physical IFC occurrence or a synthetic placeholder. The label identifies the design, not necessarily the occurrence. IFC IDs include the assembly reference and derived coordinates but are not namespaced by source file. Some TRE-derived IDs include a random suffix and are not stable between runs. |
| `CarriedTruss` | The carried occurrence in a specific girder group. Contains `localX` (along the girder, interpreted as inches), spacing, reactions in pounds, and optional bearing side, hanger angle, and governing durations. Uplift from load-case extraction is negative or zero; SST receives a non-negative magnitude. |
| `HangerResultsMap` | Candidate arrays keyed by `girderId::carriedInstanceId` for the active job. The app resets it on a new analysis. An empty stored array means a successful lookup returned no candidates; a missing key does not say whether the connection was skipped, not queried, or failed. |
| `ParsedInventory` | Parsed XML entries and a normalized model-name set used for availability. One inventory is shared by the selector and summary and stays loaded when a job changes. |

The sample is analyzed on mount. The import UI stages multiple IFC/TRE files and invokes `analyzeFiles(ifcFiles, treFiles, log)` on demand. Running with neither file type staged uses a separate inline mock input. On a completed import, the shell replaces the groups and selects the first girder if present. The parser logs read/parse failures per file and continues, so an empty result is not necessarily a clean “no girders” result.

## 4. Analysis rules and precedence

### 4.1 Read and associate inputs

1. **Read TRE first.** `parseTre` derives an uppercase label from the filename, stripping `.tre`/`.txt` suffixes. It reads MiTek sections for members, cutting information, span/pitch, end reactions, and hanger loading. Girder classification uses TRE flags, girder type text, and hanger information. `findTreForLabel` tries exact and case-insensitive matches, then the base label before `-`.
2. **Read IFC second.** The regex-based parser indexes IFC entities and identifies element assemblies. It walks referenced geometry to derive bounding boxes and centroids from truss-member points; MiTek property sets can supply left-end coordinates or lumber specifications. Assembly labels are matched to TRE for classification. Missing geometry can leave an instance without a bbox and yield a fallback centroid. This is an exporter-specific reader, not a general IFC geometry or unit-normalization library.
3. **If there are no parsed IFC instances and there are TRE profiles,** create one synthetic instance per profile. This can happen when no IFC files were supplied or when the inputs produced no usable instances.

### 4.2 Form girder groups

| Path | Current behavior |
| --- | --- |
| Spatial candidates | With at least two usable bounding boxes, scan pairs. Require opposite dominant XY axes, overlapping XY boxes with 1-inch padding on each box, and at most 12 inches difference in bottom Z. Require a girder/non-girder pair and check the carried centroid against the girder's dominant-axis bounds with a 12-inch buffer. Carried labels generally must repeat (or match a repeated base after stripping `GE`); models with fewer than ten instances bypass that rule. Deduplicate carried instances by ID per girder. |
| Positioning with TRE hanger rows | Where a girder has `LG*T` entries, build connections from those entries. Match each to an available IFC candidate by exact or base label after spatial ordering; create a synthetic occurrence when no match exists. Use the TRE offset and hanger angle. A bottom-chord member name may be matched by its X-coordinate range. Same-position hanger rows can remain distinct. |
| Positioning without hanger rows | For a spatial group, derive the offset from the carried centroid along the girder axis and infer orientation from box dimensions. Sort by along-girder offset and calculate spacing as the difference between successive positions. |
| Group filtering and fallback | Spatial groups must contain 2–20 carried rows. If any spatial groups survive, return those groups (T07 first, otherwise larger groups first); there is no fallback pass to add other girders. If none survive, fall back to instances classified as girders: use hanger rows when available; otherwise use label-based candidate sets and schematic offsets, typically 24 inches apart after the first position. This fallback can use all non-girders and has sample-specific label rules. |

### 4.3 Resolve bearing reactions

1. The primary path takes the bearing location from the girder TRE `LG*T` field `[16]` and uses the **carried** TRE `REACTION INFO`. The adjusted target is `bearingLocation - leftStub` when nonnegative; otherwise it is `span - bearingLocation - rightStub`. The code does not retry the other adjustment if matching fails.
2. Match a load-case header bearing and its total reaction rows (`loadType = -1`) using the implemented **4.1-inch** tolerance. Bearing side is inferred from the matched coordinate relative to the header's outer coordinates. Keep the largest reaction as downward demand and the smallest negative reaction as uplift (or zero if none is negative). Retain the duration-of-load factor of each governing case when available.
3. If the primary path does not produce a result, infer the carried end from its bounding-box endpoints relative to the girder center. Use left when geometry is missing, then read `Reaction1/2` and `Max Uplift1/2` for the inferred end. Missing fallback values default to zero.

**Known baseline risks:** Connection positioning sorts hanger entries by offset, but enrichment indexes the original hanger array. Unsorted TRE rows can therefore associate a bearing with the wrong connection. A missing/unparseable hanger bearing offset can normalize to zero, which is also a valid coordinate. Synthetic geometry, missing loads, and true zeros are not consistently distinguishable downstream. Some older bearing notes describe different tolerances or rules; use the current code for behavioral regression and signed-off fixtures for engineering correctness.

## 5. Hanger lookup and inventory logic

### 5.1 Build and submit one lookup

- Build the default SST payload from the girder and carried connection. Girder dimensions use the largest parsed bottom-chord cross-section; carried width uses its bottom chord and carried depth uses heel height at the bearing end, with fallbacks. Map lumber species and ply from available profile fields, and detect a vertical structural web near the connection for king-post dimensions.
- Convert reactions to rounded absolute pounds in the request. Derive skew from hanger angle and slope from the carried bottom chord at its bearing. Translate governing duration factors to SST enum values, with roof/wind defaults. The default map uses IRC 2018; the implemented ANSI/TPI classification uses a strict `< 5 × bottom-chord depth` comparison against the nearest stub-adjusted girder bearing.
- The current selector applies overrides for style, fastener type, ANSI/TPI classification, and load durations. It auto-queries when the output panel mounts or the effective payload changes; it also offers a manual query action. Result table filters are client-side and do not constrain what the summary receives.
- `submitToSST` reads the token, posts the payload, and normalizes candidate models, downward/uplift capacities, dimensions, MSRP, and installed cost index. Missing/invalid credentials, HTTP failures, and network failures surface as errors; a successful empty result is not a transport error. Browser fetch errors cannot reliably distinguish CORS from other network failures.

### 5.2 Batch lookup

- `JobSummaryTable` starts a batch when mounted and offers a manual trigger. It takes connections with positive downward reaction and no entry in the result map. A stored empty array is not queued again. It uses the default payload for each queued connection, not the selector's temporary overrides.
- Requests are sequential with a delay of about 800 ms between them. Authentication failure stops the batch; other failures are counted. Stop/unmount prevents subsequent requests, but does not abort a fetch already in flight. Successful results are written to the shared map.

### 5.3 Inventory and ranking

- Require a `HangerInventoryXml` root and at least one `InventoryList > Inventory` entry. Match `MaterialName` to the hanger model after trimming, uppercasing, and collapsing whitespace. A model is available if any matching entry is not flagged `IsUnavailable`. Price and special-order flags are read but do not change the availability test or order.
- Select the lowest positive installed cost index among in-stock candidates if any; otherwise use the same ranking among all candidates. A zero or missing index ranks as unknown (after known positive values); ties keep the first returned candidate. No candidate means no selection. The schedule's displayed cost total sums the selected candidates' `cost` values (SST MSRP); it is not an installed-cost quote or an inventory-price total.

## 6. Summary and workbook contract

- Derive **one schedule row per carried connection**, including unresolved rows. Group rows by girder for display; count selected models and count how many selected rows reference an available model. Availability flags are not stock quantities and are not decremented.
- The Excel workbook is generated from current summary rows and contains **Hanger Schedule** (girder, carried label, offset, model, capacities, dimensions/bearing, MSRP, stock, selection mode) and **Model Summary** (model, quantity, in-stock count, totals). Unresolved rows remain in the schedule with placeholder data. Existing workbook ordering and field meanings are compatibility constraints for the UI rework.
- Current result caching does not record the payload or overrides that produced a candidate list. The summary can therefore combine manually overridden single-query results with default batch results. Results may arrive after switching views or replacing a job because there is no job-generation or request-version check at the write boundary.

## 7. Proposed application-layer contract for the rework

These changes support the [UI spec](ui-rework-spec.md); they do not replace parser, mapper, ranking, or workbook rules.

| Area | Proposed responsibility |
| --- | --- |
| Job controller | Own staged inputs separately from `activeJob`. A run has an ID and source (`sample` or `import`). Show diagnostics from the run. Commit a completed run atomically; preserve the prior job on a failed run. Define partial import and failure using attributable parser diagnostics rather than assuming zero groups means failure. If the current parser cannot expose the distinction reliably, agree on a minimal diagnostics contract before implementation. |
| Identity and async writes | Scope connection keys by job generation and disambiguate file-local instance IDs at the application boundary as needed. Attach a request ID and effective-payload fingerprint to each lookup. Reject stale completions from an old job or superseded request. Do not persist unstable parser IDs as long-term identifiers. |
| Lookup controller | Expose explicit single and batch commands; neither view mounting nor input edits should send requests. Track `not queried`, `running`, `found`, `no match`, `failed`, and `not eligible for batch` distinctly; retain completed results when stopped. When effective inputs change, mark the matching cached results stale until a successful refresh. Keep the last successful result distinguishable from the latest attempt. |
| Derived selectors | Compute result counts, ranking, and schedule rows once from the active job, lookups, and inventory. Have the summary and exporter use that same snapshot. Make it clear when an overridden lookup supplies schedule data; decide before implementation how stale candidates are represented in an otherwise unchanged workbook. |
| Presentation and adapters | Keep diagram sizing and view mode local to components; keep job and connection selection in shared state. Separate the SST transport, inventory read, and workbook generation from rendering. Expose controlled errors to the UI without logging credentials or unnecessarily logging raw project-derived payloads. |

```text
stage files → run analysis → commit job OR keep prior job + show failure
active job + selected connection → map inputs → explicit lookup → record outcome
active job + pending eligible rows → explicit batch → record outcomes incrementally
active job + outcomes + inventory → derived schedule snapshot → display / Excel
```

**Domain changes require separate approval:** correcting hanger-row alignment, changing bearing tolerances, broadening IFC support, changing fallback or batch eligibility rules, and redefining an engineering zero are not part of a presentation refactor. Surface known provenance where the current data supports it; do not label untraceable values as verified.

## 8. Verification plan

| Area | Regression or acceptance check |
| --- | --- |
| Parser and domain data | Capture baseline outputs for the bundled sample, `Test3/`, and `Test4/`: girder labels, ordered connection labels and offsets, bearing sides, reactions, and governing durations. Compare semantic fields, not random instance IDs. Add fixture coverage for repeated labels, equal offsets, no IFC instances, missing reaction data, and malformed input. |
| Source precedence | Verify that available `LG*T` positions override inferred offsets and loaded `REACTION INFO` totals override fallback end reactions. Use domain-reviewed fixtures to distinguish a baseline regression from a known baseline defect. |
| Payload and adapter | Snapshot representative default and overridden payloads. Stub SST for successful results, empty responses, authorization failures, and network failures. Verify that negative uplift becomes a positive API magnitude without changing the domain value. |
| State and sequencing | Verify no background requests on view entry or input edits in the proposed flow; retry, stop-after-current, change connection, replace job during an in-flight request, and prevent old or superseded results from updating the active view or summary. |
| Inventory and export | Cover duplicate XML model names, invalid XML, missing inventory, tied/unknown cost indexes, and fallback when none is in stock. Compare sheet names, headers, worksheet rows and totals, including partial schedules. |
| Build | `npm run lint` type-checks and `npm run build` bundles. There is no automated test script in the current package; the checks above define coverage to add or run during implementation. |
