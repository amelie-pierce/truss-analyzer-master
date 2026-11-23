# Truss Analyzer — UI rework specification

**Status:** Draft for review  
**Scope:** React UI and interaction state. No changes to structural analysis, hanger selection rules, or export format.

## 1. Purpose

Make it easy for a truss designer or estimator to go from project files to an Excel hanger schedule, while making it clear which job, girder, and connection they are looking at and what still needs review.

Primary workflow: **Import files → Run analysis → Inspect girders and connections → Find hangers → Review and export the job summary.**

Success means a first-time user can complete that workflow without guessing which icon to click, whether the results are from the sample or their own files, or whether a missing hanger is pending, unavailable, or the result of an error.

## 2. Current UI and what to fix

| Today | Rework |
| --- | --- |
| A bundled job loads on startup; running with no inputs can run a separate mock job. | Start with an empty workspace. Offer one explicit **Load sample job** action using the bundled files, and label the sample throughout the app. |
| File controls show counts but not filenames; analysis diagnostics are collected but not shown in the main UI. | Show queued filenames, distinguish queued files from active results, and surface warnings and errors with expandable diagnostics. |
| Fixed-width columns, very small uppercase labels, and mixed light/dark panels compete for space. | Use one visual system, a flexible layout, and readable table and diagram labels. |
| The hanger selector is opened by an icon-only action; some status labels imply verification even when none occurred. | Use a labeled **Find hangers** action, keep connection context visible, and derive status from actual workflow state. |
| Opening the Job Summary can trigger SST requests, and the current batch action skips connections already queried. | Opening the summary should be read-only. Provide an explicit batch action whose label reflects its scope. |

## 3. Shared layout and visual direction

- **App header:** Truss Analyzer, active job name (derived from the IFC filename or shown as **IFC/TRE job** if ambiguous), **Analyzer** and **Job Summary** navigation, and **Settings**. Show **Sample job** when applicable. Show the active job independently of any queued replacement files.
- **Desktop (1280 px and up):** Girder navigation rail (about 240 px) plus a flexible workspace. On the Analyzer, give the diagram and connection table priority; place properties below or in an expandable panel rather than allocating a permanent third column. Give the Job Summary the full content width.
- **Below 1280 px:** Stack diagram, connection table, and properties. Below 768 px, replace the navigation rail with an accessible girder picker. Horizontal scrolling is limited to wide tables and diagrams; the page itself must not scroll sideways.
- **Style:** Light neutral surfaces, one blue accent for selection and primary actions, subtle borders, and semantic colors for warnings and errors. Preserve distinct diagram marks for members and loads, with labels or line styles so color is not the only cue.
- **Typography:** Aim for 14–16 px for general text and 13–14 px for dense table text. Use sentence case, tabular numerals, right-aligned numeric columns, and visible units. Prefer **Downward reaction (lb)** to **Rxn ↓** and define **Duration of load (DOL)** at first use. Distinguish connection demand from hanger capacity.

## 4. Screen requirements

### 4.1 Job setup

1. Initial view offers **Import project files** and **Load sample job**. Do not run analysis or send SST requests on page load.
2. Show separate multi-file pickers for IFC and TRE. Display filenames and counts; allow removing files and replacing each selection before running. **Run analysis** is disabled when neither group has a staged file. Recommend using both formats; preserve current single-format behavior.
3. Show a running indicator and prevent duplicate submissions. Show per-file warnings and errors when available. Clearly distinguish analysis failure, analysis with warnings, and successful analysis with no girders found. When TRE-only or unusable IFC input causes the existing parser to build a model from TRE data, label that fallback explicitly rather than presenting it as confirmed IFC geometry.
4. Staging new files must not replace the active job. A failed run leaves prior results visible with a message explaining which job is still active. A completed run replaces the displayed dataset, even if it found zero girders, and resets connection selection and hanger results. Keep the existing shared inventory behavior and show which inventory file is active.

### 4.2 Analyzer

- Show a searchable girder list with girder labels and carried-truss counts. Select the first girder after a successful run with results. Keep the girder and connection selections when switching tabs within the same job.
- Show the selected girder label, span, and **Plan** and **Side elevation** diagrams. Selecting a carried truss highlights it in both diagrams. Give diagrams accessible names and legends; make connection values available in the table without relying on the diagram.
- Show a carried-truss table with truss label/ID, offset, downward reaction, uplift, duration of load, and a labeled **Find hangers** action. Use inches or feet/inches for dimensions and pounds for loads; distinguish unavailable values from a confirmed zero. If the existing presentation uses a fallback or schematic value, identify it as such where the source is known. Do not change solver output or substitute defaults silently.
- Keep existing engineering and material details accessible in a **Properties** section. If a girder has no carried trusses, show that state in the table area and leave girder properties and diagrams available.

### 4.3 Hanger selector (SST)

- Open the selector for one specific girder-to-carried-truss connection. Show both labels, load demands, and a **Back to Analyzer** action that restores the prior selection. Changing the selected connection must not display results from the previous one.
- Keep existing editable settings, mapped values, **Reset to defaults**, results filters, and sorting. Separate **Connection inputs** from **Hanger results**. Use an explicit **Search SST** action rather than submitting on view open or field edits. If inputs change after a search, mark the displayed results **Out of date** until the next successful search.
- Show loading, no matches, no matches for the current filters, and request failure as distinct states. Missing or rejected credentials should link to **Settings → SST access**. A saved token means **Token saved**, not **Connected**. Keep the token masked and out of diagnostics and exports. Analysis and diagram review must work without it.
- Allow importing/removing the existing XML inventory, show import errors, and identify inventory status in results. Without an inventory file, use **Inventory not loaded**, not **Out of stock**. Clearly distinguish candidate hanger capacity from connection demand and avoid wording that implies engineering approval.

### 4.4 Job Summary

- Show girder and connection counts, selected-hanger count, unresolved count, existing cost totals, and inventory status. Keep the model quantity breakdown and schedule grouped by girder. Explain the existing selection rule: prefer the lowest installed-cost in-stock option when available; otherwise use the lowest installed-cost overall. Do not introduce manual selection or change ranking in this rework.
- Use **Find pending hangers** to start lookups for eligible connections without successful results, using the existing eligibility rules. Show progress, a **Stop after current request** action, and distinct outcomes for not queried, no match, failed, and hanger found. Retain completed results when stopped; allow retrying failures. Connections not eligible for batch search should be labeled, not left looking perpetually pending.
- Keep **Export Excel** available for a non-empty schedule. If any connections lack a hanger or have stale results, show that state before download and make clear that the schedule is partial. Preserve the existing workbook schema and sheets: **Hanger Schedule** and **Model Summary**.

## 5. Accessibility and feedback

- Use semantic buttons, labeled fields, and real table headers. Make girder and carried-truss selection operable by keyboard, with visible focus indicators and an accessible selected state.
- Give icon-only utilities accessible names. On opening or closing settings or the hanger view, move focus to the new context and return it to the initiating action on close when possible.
- Announce analysis and lookup completion, warnings, and errors via appropriate live regions without announcing every progress tick. Keep error messages visible near the action until resolved or dismissed.
- Target WCAG 2.2 AA contrast (4.5:1 for normal text, 3:1 for large text and meaningful UI boundaries); validate keyboard-only use, 200% zoom, and reduced-motion behavior.

## 6. Implementation boundaries

- Use existing React, TypeScript, Tailwind, and icon dependencies. Define reusable surface, text, status, spacing, and focus styles in `src/index.css`.
- `src/App.tsx`: job source, staged vs. active job state, selected girder/connection, shared inventory and hanger results. Associate in-flight SST results with the originating job and connection; ignore responses from an old job after a replacement run.
- Update the active screen components in `src/components/`: `FileUpload`, `GirderList`, `GirderDetails`, `SSTWorkspace`, and `JobSummaryTable`. Adapt `LogPanel` for expandable diagnostics. The unused JSON export view is not part of this rework.
- Preserve `src/lib/parser.ts`, SST payload mapping, API contract and token storage mechanism, inventory parsing, and workbook schema. No new solver, proxy, accounts, persistent job history, or deployment changes. Any question about analysis accuracy or the meaning of a source field goes to a domain reviewer rather than being decided in the UI code.

## 7. Acceptance checks

| Scenario | Pass condition |
| --- | --- |
| First visit and sample job | The empty state shows import and sample actions. The sample loads only when selected, stays visibly labeled, and is never substituted for a user upload. |
| Real job import and replacement | Test with the bundled sample and at least one IFC/TRE set from `Test3/` or `Test4/`. Girder and connection counts and calculated values agree with the baseline for the same files. Queuing files or a failed run does not hide prior results; completing a new run does not carry over old hanger results. |
| Analysis edge cases | No input, invalid/unreadable files, valid inputs with no detected girders, and the TRE fallback are distinguishable. Diagnostics are reachable from the affected screen. |
| Connection review | Selecting a connection updates both diagram highlights and the visible context. Returning from the selector or switching tabs preserves the selection within the same job. |
| Single and batch SST | With a stubbed SST response, verify success, no matches, auth failure, network failure, retry, cancellation, and late responses after job replacement. Visiting a view or editing an input does not issue a request. |
| Inventory and export | Verify valid and invalid XML imports, removal, and shared inventory state. Download a partial and a completed workbook and verify sheet names, column order, row counts, and selection values against equivalent baseline results. |
| Layout and accessibility | Test at 1440, 1024, and 390 px and at 200% zoom. Complete import, selection, lookup, and export by keyboard with visible focus; no page-wide horizontal overflow. |
| Build | `npm run lint` and `npm run build` pass after implementation. |

## 8. Delivery order

1. Shared styles, app shell, navigation, job setup, and diagnostics.
2. Analyzer layout, selection behavior, and diagram readability.
3. SST selector, inventory feedback, batch status, and Job Summary.
4. Fixture regression, accessibility review, and workbook comparison.

**Review before implementation:** Confirm the proposed light, desktop-first direction, the explicit sample entry point, and replacing automatic SST lookups with user-initiated requests. Confirm with a domain reviewer which material and load values should carry fallback labels.
