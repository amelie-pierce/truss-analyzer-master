# Truss Analyzer — Parser and data specification

**Status:** Draft for review  
**Scope:** A field-level reference for IFC/TRE reading, connection inference, and reaction resolution. Behavioral overview is in the [technical specification](technical-spec.md). Values described here are current code behavior, not an engineering approval.

## 1. Input assumptions

- IFC inputs are text files, typically STEP-style entity lines such as `#123=IFCELEMENTASSEMBLY(...);`. The reader is regex-based and exporter-specific (MiTek); it is not a full IFC engine.
- TRE inputs are MiTek text reports. They may be uploaded as `.tre`; bundled samples use `.tre.txt` and are read as text.
- A TRE profile describes a truss *design*; an IFC element assembly describes a *physical occurrence*. The same label can occur many times.
- Units used by the code are inches and pounds. IFC coordinate values are consumed as-is; there is no global unit normalization.

## 2. TRE reader

### 2.1 Label and profile identity

`parseTre` derives the label from the filename by stripping trailing `.tre` or `.txt` suffixes (twice) and uppercasing it. `findTreForLabel` resolves a truss instance to a profile by exact match, case-insensitive match, then base label before `-`. If multiple files produce the same key, the last one parsed wins. There is no file hash or variant dimension in the key.

### 2.2 Derived fields

| Field | Source / behavior |
| --- | --- |
| `isGirder` | Set from `TRUSS TYPE=...girder`, `GIRDER=YES`, girder wording in `TRUSS INFO` / `[ADDITIONAL TRUSS INFO]`, or a populated `[Hanger Loading Info.]` section. |
| `span`, `pitch` | `ROOF BASICS` value line; default zero. |
| `members` | Parsed from member-info blocks: name, type, size/grade/species, width/depth, coordinate list, structural flag. |
| `cuttingMembers` | `[ADDITIONAL CUTTING INFO]`; treated as authoritative for grade/size exceptions. |
| `reactions` | `Reaction1`, `Reaction2`, `Max Horz1`, `Max Uplift1`, `Max Uplift2` from additional truss info; defaults zero when absent. |
| `leftStub`, `rightStub`, `leftHeel`, `rightHeel`, `ply`, `bottomChordSlopes` | Read from additional truss info when present. |
| `dol` | Parsed from load-case data; null if not found, with no invented default at this stage. |
| `hangers` | Each `LG*T` row in `[Hanger Loading Info.]`. See field map below. |
| `rawText` | Retained for `REACTION INFO` extraction. |

### 2.3 `LG*T` hanger row (fields after `=`)

| Index | Meaning used by code |
| --- | --- |
| 2 | `xInches`, the position along the girder from its left end. |
| 4 | Carried label. |
| 5–6 | Width and heel height. |
| 12–13 | Skew/bearing flags; field 13 is read as a bearing-side flag but is not the primary source for reactions. |
| 14 | Hanger angle (carried relative to girder); default 90. |
| 16 | Bearing location of the carried truss, in inches; default 0 when missing or unparseable. |

## 3. IFC reader

1. Index every `#id=ENTITY(...)` line into a map.
2. Find `IFCELEMENTASSEMBLY` entities; each becomes a potential truss occurrence. Extract a short quoted label using the heuristic in `extractLabel`.
3. Follow entity references from each assembly to geometry, tracking whether the path passes through an `IFCMEMBER` whose text contains `Truss Member`. Collect `IFCCARTESIANPOINT` coordinates only on those paths; skip plates and non-geometry entities such as owner history and styling.
4. From collected points, compute an axis-aligned bounding box and centroid. If a MiTek `MiTek_PSet_LeftEnd` property set is linked to the assembly, use its X/Y/Z as the left end and as a fallback centroid source.
5. When geometry is unavailable, derive a deterministic fallback centroid from the label and assembly ID. These coordinates are illustrative, not measured.
6. If a linked MiTek material property set names a top chord, bottom chord, or web, copy that specification to the instance for display.
7. Tag `isGirder` from the matched TRE profile where possible.

## 4. Girder/connection inference

### 4.1 Inputs

`analyzeFiles` reads TRE profiles first, then IFC occurrences. If no IFC instances resulted but TRE profiles exist, it creates one synthetic instance per profile using `MOCK_<label>` IDs. All groups are built by `analyzeConnections`.

### 4.2 Spatial path

Used when at least two parsed instances have bounding boxes:

1. For each pair, require one truss to be predominantly X-aligned and the other predominantly Y-aligned.
2. Require XY bounding boxes to overlap with a 1-inch buffer on each box and bottom elevations within 12 inches.
3. Determine carrier/carried using TRE classification (or IFC flag) and centroid containment along the girder's dominant axis with a 12-inch buffer.
4. Require the carried design to be repeated in the model, with a special allowance for `GE` variants and for models under ten instances.
5. Group by girder, discard carried instances that are themselves carriers, and build ordered connection rows.

### 4.3 Connection positioning

- **With girder hanger rows:** sort rows by `xInches`; order candidate IFC instances along the girder's dominant axis; reverse if the geometry suggests the first known instance aligns with the far hanger. Match each hanger to an exact or base label, then synthesize an occurrence if none matches. Offset comes from the hanger row; spacing is computed later.
- **Without hanger rows:** derive each offset from the carried centroid along the girder axis, infer orientation from bounding-box dimensions and side from the cross-axis centroid. Sort and derive spacing from successive offsets.
- **Fallback girder path:** if no spatial group survives, use TRE-classified girders and, when hanger rows exist, use the same positioning routine. Otherwise use label-based candidate sets and schematic positions (commonly 24 inches apart after the first).

### 4.4 Group selection rules

- Spatial groups with 2–20 carried rows are accepted; if any exist, they are returned, sorted with T07 first and then by descending carried count. The fallback path is not merged in.
- Same-position hanger rows are retained. Earlier filters that dropped repeated offsets or label variety were removed to support girders such as T09 and CJ08.

## 5. Reaction and bearing resolution

For each carried row, `enrichCarriedTrusses` first attempts `parseReactionAtBearing`:

1. Find the `REACTION INFO` section and iterate load-case blocks.
2. Parse each header's bearing coordinates and duration factor. Match the target bearing using a **4.1-inch** tolerance; infer left/right from the smallest/largest matched coordinate set.
3. In the block's data lines, keep total rows (`loadType = -1`) whose bearing coordinate matches the selected bearing.
4. Downward reaction is the maximum collected value; uplift is the minimum value only when negative, otherwise zero. Store the duration factor of each governing case.

The target bearing is derived from the girder hanger's field 16: use `bearingLocation - leftStub` when nonnegative, otherwise `span - bearingLocation - rightStub`. Only one adjustment is attempted.

**Fallback:** infer the physical carried end from bounding-box endpoints relative to the girder center (left when geometry is missing), then read the corresponding left/right reaction and uplift. Missing endpoint values become zero.

## 6. Data quality and known limitations

| Issue | Why it matters |
| --- | --- |
| Hanger rows sorted for positioning but indexed in original order for reaction enrichment | Unsorted source rows can assign a bearing/reaction to the wrong carried row. Verify with fixtures. |
| Bearing location zero is both a valid coordinate and a missing-value default | A missing field can silently select a bearing at the truss end. |
| Last TRE profile wins for duplicate labels | Variant files with the same derived label can overwrite each other. |
| Fallback centroids and positions are synthetic | Downstream geometry and angles can look authoritative without a measured source. |
| Separate bearing notes use different tolerances/rules | The current code is the regression baseline; older docs may describe superseded logic. |
| Spatial path suppresses fallback groups once any spatial group survives | Some TRE-classified girders may not appear in mixed-quality jobs. |
| IFC parser consumes coordinates without unit normalization | Files exporting feet/meters or different coordinate systems may be mis-scaled. |

## 7. Extension guidance

- Keep format-specific parsing separate from inference rules so a fixture can fail at a named stage.
- Introduce a provenance field per derived value (source file, section, rule, confidence) before changing any fallback; this supports the provenance labels in the UI spec.
- When changing a tolerance, classification rule, or fallback, add a signed-off fixture and update the semantic baseline described in the [testing and QA spec](testing-qa-spec.md).
- Do not silently introduce new default loads, spans, or materials. Make defaulted values inspectable in the UI before changing downstream payloads.
