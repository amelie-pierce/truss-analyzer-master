# Truss Analyzer

Browser app that reads IFC and MiTek TRE files, finds girder trusses and the trusses they carry, draws plan and elevation diagrams, looks up Simpson Strong-Tie (SST) hanger candidates, and exports an Excel hanger schedule.

This README is written for coding agents and people alike. Every command and result below was run against this commit.

## Setup prompt

Paste this into a coding agent to set the project up:

```text
Set up the Truss Analyzer repository for local development.

1. Clone https://github.com/amelie-pierce/truss-analyzer-master and work in the repository root.
2. Read README.md fully before running anything. Then read docs/technical-spec.md and docs/ui-rework-spec.md.
3. Check that Node.js is version 20 or later. If it is not, stop and tell me.
4. Run `npm ci`.
5. Run `npm run build`. It must pass.
6. Run `npm run lint`. It is expected to fail with exactly the 4 errors listed in README.md under "Known state". Report any other error.
7. Start `npm run dev` and confirm that http://localhost:3000 answers with HTTP 200. If port 3000 is already in use by this app, reuse that server; do not start a second one.
8. Do not change code, install or upgrade packages, commit, push, or deploy. Do not ask me for an SST token; the app runs without one.
9. Report: Node version, the result of each step, the local URL, and anything that differs from README.md.
```

## Facts

| Item | Value |
| --- | --- |
| Type | Single-page app, client only. No backend, router, state library, or database |
| Stack | React 19, TypeScript 5.8, Vite 6, Tailwind CSS 4 |
| Node.js | 20 or later (`engines` in `package.json`) |
| Package manager | npm, with `package-lock.json` |
| Dev URL | `http://localhost:3000` |
| Environment variables | None required. `PORT` sets the port of `npm run start` (default 3000). `DISABLE_HMR=true` turns off hot reload in dev |
| Secrets | None in the repo. The SST token is typed into the app and kept in browser `localStorage` |
| Tests | None. No test script and no CI |
| Deploy | Railway, configured in `railway.json` |

## Commands

| Command | What it does | Expected result |
| --- | --- | --- |
| `npm ci` | Installs the locked dependencies | Passes |
| `npm run dev` | Dev server on port 3000, all interfaces | Serves the app with the bundled sample loaded |
| `npm run build` | Production build into `dist/` | Passes, with a warning about a chunk larger than 500 kB |
| `npm run lint` | Type check (`tsc --noEmit`); there is no other linter | **Fails with 4 errors**, see below |
| `npm run preview` | Serves `dist/` locally | Needs a build first |
| `npm run start` | Serves `dist/` on `$PORT`; used by Railway | Needs a build first |

## Known state

- `npm run lint` fails with 4 errors. Do not treat them as caused by your change:
  - `src/App.tsx` lines 15–17: three `?raw` imports have no type declaration (TS2307).
  - `vite.config.ts`: `preview.allowedHosts: true` does not match the Vite type (TS2769).
- The app loads and analyzes the bundled sample job on every page load. The sample files are compiled into the bundle, which is why the build warns about chunk size.
- These files are not used by the running app: `src/components/HangerPanel.tsx`, `src/components/ExportTab.tsx`, `src/data.ts`. `src/components/LogPanel.tsx` is imported but never rendered.
- The `motion` package is installed and never imported.
- Several on-screen status labels are fixed text and not derived from data. `docs/ui-rework-spec.md` lists what to fix.

## Repository map

| Path | Role |
| --- | --- |
| `src/main.tsx`, `src/App.tsx` | Entry, shell, tabs, shared state, sample loading |
| `src/components/FileUpload.tsx` | File pickers, run button, SST token box |
| `src/components/GirderList.tsx` | Girder list in the sidebar |
| `src/components/GirderDetails.tsx` | Carried-truss table, properties, both SVG diagrams |
| `src/components/SSTWorkspace.tsx` | Hanger selector: inputs, results, inventory |
| `src/components/JobSummaryTable.tsx` | Job summary, batch lookup, ranking, Excel export |
| `src/lib/parser.ts` | IFC and TRE parsing, girder-to-carried-truss inference. No UI code |
| `src/lib/sst-mapper.ts`, `sst-api.ts`, `sst-types.ts` | SST request payload, client, types |
| `src/lib/inventory.ts` | Inventory XML parsing |
| `src/types.ts` | Shared types |
| `src/index.css` | Tailwind entry and global styles |
| `2214224-08T.ifc.txt`, `t02.tre.txt`, `t07.tre.txt` | Bundled sample job: 1 girder (T07) carrying 8 trusses |
| `Test3/`, `Test4/` | Larger IFC and TRE fixture sets for manual checks |
| `docs/` | Specifications and calculation notes |

## Documentation

Read in this order:

1. [Technical specification](docs/technical-spec.md): system boundary, layers, data flow, business rules.
2. [UI rework specification](docs/ui-rework-spec.md): workflows, layout, acceptance checks.
3. Calculation notes, when touching the parser: [reaction calculation](docs/reaction-calculation.md), [bearing calculation](docs/bearing-calculation.md), [TRE-based bearing detection](docs/tre-based-bearing-detection.md), [legacy IFC bearing detection](docs/legacy-ifc-bearing-detection.md).

## Rules for agents

- Keep UI work out of `src/lib/`. Do not change parser output, the SST payload, hanger ranking, or the Excel workbook format unless the task says so.
- Questions about analysis accuracy or the meaning of a source field go to a person. Do not decide them in code.
- Never show a status, badge, or value that is not derived from job data, a request result, or user input. Show a missing value as missing.
- Never put an SST token in code, logs, commits, exports, or chat.
- A live SST lookup needs a token from a person and sends data to an external service. Do not run one without being asked.
- Before you report a change as done, run `npm run build` and `npm run lint`, and say which lint errors existed before your change.
- Do not merge, push to `main`, or deploy without approval.

## External service

Hanger lookups go from the browser to `POST https://api.strongtie.com/gws/hanger-selector/hangers` with a bearer token. Analysis, diagrams, and the job summary work without a token; only hanger results need one.

## Deploy

Railway builds with `npm install && npm run build` and starts with `npm run start` (`railway.json`). Connect the repository in Railway to deploy.
