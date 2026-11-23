# Truss Analyzer — Testing and QA specification

**Status:** Draft for review  
**Scope:** Verification strategy for the analyzer, SST integration, inventory, schedule export, and UI workflows. This spec defines coverage and acceptance gates; it does not change application behavior.

## 1. Current state and risk

- `npm run lint` is a TypeScript type check (`tsc --noEmit`) and `npm run build` is a Vite build. There is **no automated test script** and no test framework in `package.json`.
- Analysis correctness depends on regex parsing of exporter-specific IFC/TRE text, multiple fallbacks, and inferred geometry. A change that compiles can still silently change reactions or connection ordering.
- SST calls use a bearer token from `localStorage`, may be blocked by CORS, and can succeed with an empty candidate list. These are different outcomes that tests must distinguish.
- No stable parser IDs: some synthetic IDs contain random values. Tests must compare semantic data, not generated IDs.

## 2. Test pyramid

| Level | Target | Suggested tooling | What it catches |
| --- | --- | --- | --- |
| Unit | `src/lib/*.ts`: TRE field parsing, IFC entity walking, connection inference, reaction extraction, payload mapping, SST response normalization, inventory parsing, ranking | Vitest with local fixture strings | Field and enum regressions, negative/zero confusion, fallback precedence |
| Component / integration | Import controls, girder list, carried table, selector, summary, focus and keyboard flows | Vitest + React Testing Library, mocked file reads and SST adapter | State wiring, stale selections, request triggers |
| End-to-end | Full workflow in a browser with a mocked SST endpoint; separate live smoke test with a real token | Playwright (dev server at `localhost:3000`) | Navigation, upload, download, CORS/token failures |
| Manual / domain | Engineering sign-off on reaction and bearing values for signed-off fixtures | Signed baseline sheet | Semantics a passing unit test cannot establish |

Adding a test framework is a tooling change that should be approved before implementation. Until then, the manual checklist in section 6 is the release gate.

## 3. Fixture set

| Fixture | Use |
| --- | --- |
| Bundled `t07.tre.txt`, `t02.tre.txt`, `2214224-08T.ifc.txt` | Default demo baseline; also verifies sample labeling after the UI rework. |
| `Test3/` and `Test4/` IFC + TRE sets | Real-world variety: repeated labels, GE variants, multiple hangers, girder/jack combinations. Pin exact fixture versions in test data. |
| Synthetic minimal IFC | Two perpendicular assemblies with overlapping bounding boxes and bottom elevations; verifies spatial connection detection. |
| TRE with hanger rows | `LG*T` lines with known offsets, angles, and bearing locations; verifies positioning and reaction mapping. |
| Edge-case fixtures | No IFC instances, malformed TRE text, missing `REACTION INFO`, missing reactions, duplicate labels, equal hanger offsets, no hangers, empty inventory, invalid XML. |
| Mock SST responses | Candidates with known capacities/costs, empty `lstHangerOutput`, status error, 401/403, HTTP 500, network/CORS-style failure. |

Store expected results as semantic snapshots keyed by fixture and semantic connection identity (girder label, carried label, position), never by generated UUIDs or random IDs.

## 4. Required test cases

### 4.1 Parser and data

- Each TRE field is read from the documented section and label derivation is correct for `.tre` and `.txt` filenames.
- Girder classification from TRE flags and hanger sections; TRE classification overrides IFC-derived `isGirder` when both exist.
- IFC assemblies yield expected bounding boxes and centroids; geometry recursion stops at depth 12 and excludes plate and non-truss entities.
- Exact, case-insensitive, and base-label TRE matching: labels with suffixes, `GE` variants, and duplicates.
- Connection detection: opposite axes, XY overlap tolerance, bottom-elevation tolerance, centroid containment, repeated-label filter, and the girder-cannot-be-carried rule.
- Hanger-driven positioning, including flipped girder orientation, matching by exact/base label, and synthetic placeholders for unmatched hanger labels.
- Fallback positioning when a girder has no hanger rows; spacing computation for sorted rows; 2–20 carried-row group limits; T07 priority ordering.
- Reaction precedence: `LG*T` bearing location plus `REACTION INFO` total rows beats the bounding-box end-reaction fallback. Verify governing downward and negative uplift values and their duration factors.
- Known risk regression: hanger rows sorted by offset but reactions indexed by original order. Add an unsorted-row fixture that documents current behavior or verifies an agreed fix.
- Zero versus missing values: a parsed zero, an absent field, an unparseable bearing location, and a TRE-only synthetic instance must not be conflated silently.
- Import failure and partial parse: one bad file does not stop other files; logs report per-file failures; empty groups are distinguishable from a read failure.

### 4.2 SST mapping, lookup, and inventory

- Snapshot the default payload: material codes by species, bottom-chord dimensions, king-post detection at the connection, ply, skew/slope signs, rounded absolute loads, durations, ANSI/TPI classification, and flush option.
- Overrides change only the intended fields; resetting overrides restores payload-derived defaults; changing the selected connection resets overrides.
- Response normalization maps model, capacities, dimensions, MSRP, installed cost index, and series; transport errors and successful empty results are distinct.
- A token absent, expired, invalid, or cleared produces the correct user state. Token values never appear in logs, diagnostics, exports, or snapshots.
- Batch: skips already-resolved connections, queues eligible ones with positive downward reaction, respects stop-unmount, handles auth abort and non-auth failures, retains successful results, and can retry failures.
- Inventory: valid XML, uppercase/whitespace normalization, duplicate models with mixed availability, missing/unavailable root, zero entries, malformed XML, and removal.
- Ranking: lowest positive installed cost in stock, fallback to lowest positive cost overall, unknown-cost ordering, tie behavior, and no candidate.

### 4.3 UI and schedule

- First visit, sample load, real import, replacement import, failed import preserving prior job, and analysis with zero detected girders.
- Girder and connection selection; both diagrams highlight the same connection; switching Analyzer/Job Summary preserves selection within a job.
- SST selector opens with the correct girder and carried labels; no request on mount or input edit; explicit search, filters, and sorting are client-side.
- Stale results after payload changes; late responses from an old job or a superseded request do not update the active view or schedule.
- Summary counts, model summary, cost total, partial-schedule warning, and Excel download with sheets **Hanger Schedule** and **Model Summary**, unchanged headers, and correct row counts.
- Layout at 1440, 1024, and 390 px, plus 200% zoom; keyboard-only completion of the main workflow.

## 5. Baseline and regression workflow

1. Freeze a baseline JSON of semantic parser output and workbook rows for the fixtures in section 3. Include fixture checksums.
2. Run every change against that baseline. Any diff must have an explicit reason: a domain-approved correction, a documented fixture update, or a reverted change.
3. For reaction-related diffs, attach a domain reviewer's confirmation. A value change is never accepted solely because tests were updated to match.
4. Keep mock SST responses deterministic; never snapshot live API output or tokens.
5. Run the accessibility checklist in [the accessibility spec](accessibility-spec.md) before release.

## 6. Manual release checklist

- [ ] `npm run lint` and `npm run build` pass on a clean install.
- [ ] Sample loads only when requested and is visibly labeled.
- [ ] Each real fixture produces the expected girder/connection counts and reviewed reactions.
- [ ] Failed and zero-result imports behave as specified and leave the prior job intact on failure.
- [ ] Single and batch lookups work with a stubbed endpoint and with a valid live token; auth and network failures show distinct states.
- [ ] Inventory import, ranking, and workbook export match the baseline; partial schedules are flagged.
- [ ] Keyboard and 200% zoom checks pass; no page-wide horizontal scrolling.
- [ ] No token, raw payload, or project text is exposed in the browser console by normal flows or test artifacts.

## 7. Definition of done

A change is complete when its unit/component coverage exists at the level above, the semantic baseline is unchanged or the diff is domain-approved, the manual checklist passes, and any new fallback or unresolved state is visible in the UI rather than silently absorbed.
