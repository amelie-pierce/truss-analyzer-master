# Truss Analyzer — Accessibility specification

**Status:** Draft for review  
**Scope:** WCAG 2.2 AA target for the existing app and the proposed UI rework. Applies to Analyzer, SST selector, Job Summary, upload controls, and diagrams.

## 1. Goal and standards

All task-critical workflows must be operable with a keyboard, understandable with assistive technology, and readable at 200% zoom. Target WCAG 2.2 AA: at least **4.5:1** contrast for normal text, **3:1** for large text and meaningful UI boundaries, visible focus, and no keyboard traps. The dense tables and technical diagrams must not be the only path to engineering values.

## 2. Core interaction requirements

| Requirement | Implementation expectation |
| --- | --- |
| Semantic controls | Use `<button>` for actions, `<label>` plus `<input>` for fields, `<table>` with `<th scope>` for data grids, and landmarks/headings for regions. Never use a styled `<div>` as a button. |
| Accessible names | Every control has a discernible name. Icon-only actions need `aria-label`; current magnifier controls need names such as **Find hangers for J06C**. Status pills and badges must have text, not color alone. |
| Focus visibility | A single clear focus style with at least 3:1 contrast against both light and dark surfaces. Do not remove outlines without a replacement. |
| Focus order | Follow the visual reading order: header navigation, file actions, girder list, selected-girder content, carried table, connector panel, summary actions. Focus moves into a newly opened selector or settings area and returns to the invoking action on close where feasible. |
| Selection semantics | Girders and carried rows use buttons or listbox/grid semantics with a programmatic selected state. Selection changes update the diagrams and the table together. |
| Live feedback | Announce analysis result, lookup completion, inventory error, and batch state changes through polite live regions. Do not announce every progress tick or every keystroke. Errors receive appropriate assertive treatment. |
| Motion and time | Respect `prefers-reduced-motion`; no auto-advancing or time-limited interactions. Long batch operations are cancellable rather than timed out by the UI. |

## 3. Screen-specific checks

### 3.1 App shell and job setup

- Header navigation identifies the current tab programmatically. The app name and active job/sample state are available as text.
- File inputs have visible labels, accepted formats, counts, and filenames. Buttons that open hidden inputs are named clearly; removing or replacing files is keyboard-accessible.
- Run status, warnings, errors, and expandable diagnostics are reachable and readable without a mouse. A failed run does not remove the active job silently.
- SST token field is a masked password field with a visible label and a saved/not-saved state that does not imply connectivity.

### 3.2 Analyzer

- Girder list is a navigable list with names and counts; the selected item is programmatically identifiable.
- Diagram panels have accessible names and text summaries of what they show. A legend identifies chords, webs, loads, and crossing lines by shape/line style or text as well as color.
- The carried-truss table has column headers, row selection, and per-row **Find hangers** actions. Numeric columns include units and use stable number formatting; a dash or **not available** is distinct from zero.
- Properties and material exceptions are exposed as text, not only as color-coded diagram labels. Selecting a carried row highlights it in the diagram but the values remain available in the table.
- On narrow layouts, the girder picker must remain reachable and the diagram must be horizontally scrollable by keyboard without trapping focus.

### 3.3 SST selector

- Connection context (girder and carried labels and demands) is announced when the selector opens.
- Input overrides are labeled fields with units or enum meanings. **Reset to defaults**, **Search SST**, view toggles, filters, and sort controls are keyboard-operable and state changes are announced.
- Result tables use headers and sortable-column semantics. Stock status is text or icon plus text. Empty, filtered-empty, failed, and no-token states are readable as text and offer recovery actions.
- No request or focus jump is triggered merely by entering a field or viewing results.

### 3.4 Job Summary

- Statistic cards expose label and value as text; badge counts are not color-only.
- Batch progress announces starts, completion, and failure without flooding: announce state transitions, not each request. Stop and retry actions remain enabled and named.
- Export produces a downloadable file and announces success or failure. The partial-schedule warning is a readable message, not only an icon.
- Grouped tables keep headers associated with their group and remain navigable at 200% zoom.

## 4. Visual and responsive criteria

- Test text and control contrast on light surfaces (`#EDEDED`, `#F2F2F2`, white) and dark surfaces (`#0C0D14`, `#12131C`, `#1E293B`). The current very small uppercase labels and low-opacity text are likely failures and need review.
- Do not convey reaction direction solely with red/blue or arrows; pair it with text. Do not rely on green/amber dots alone for status.
- Support 1440 px, 1024 px, and 390 px widths and 200% zoom. At 200% zoom, no horizontal page scroll; tables and diagrams may scroll inside their own labeled region.
- Ensure 44 × 44 px target sizes where practical for touch, or at least adequate spacing for icon actions.
- Respect reduced-motion settings for panel transitions and animated status indicators.

## 5. Audit procedure

1. Keyboard-only pass: complete import, sample load, girder selection, carried selection, single lookup, batch, inventory import, and Excel export. Record focus order and any traps.
2. Screen reader pass with NVDA or VoiceOver: verify names, roles, states, headings, table navigation, and live announcements.
3. Automated pass: run axe or equivalent on each screen state, including empty, loading, error, no-match, and results states.
4. Zoom and contrast pass at 200% and 400% zoom, plus forced-colors mode where available.
5. Diagram pass: verify that every engineering value conveyed by color or position is also available as text.

## 6. Acceptance criteria

- No critical or serious automated violations on any workflow state.
- A keyboard-only user can complete the full workflow and recover from errors without a pointer.
- All interactive elements have visible focus and accessible names; tables have headers; selected states are programmatic.
- Status, reactions, and stock information are not color-only.
- No keyboard trap, no page-wide horizontal scrolling at the tested widths, and no motion-dependent interaction.
