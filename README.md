# Truss Analyzer

Analyze IFC and TRE files to identify Girder Trusses and Carried Trusses with structural diagram visualizations.

## Specifications

- [Testing and QA specification](docs/hy4/testing-qa-spec.md) — fixtures, test cases, and release checks
- [SST integration specification](docs/hy4/sst-integration-spec.md) — API contract, token, batch lookups, and inventory
- [Parser and data specification](docs/hy4/parser-data-spec.md) — IFC/TRE fields and inference rules
- [Accessibility specification](docs/hy4/accessibility-spec.md) — WCAG targets and audit procedure
- [Release and deployment specification](docs/hy4/release-deployment-spec.md) — build, Railway deploy, and rollback
- [Technical specification](docs/hy4/technical-spec.md) — architecture, data flow, and business rules
- [UI rework specification](docs/hy4/ui-rework-spec.md) — workflows, layout, and acceptance criteria

## Tech Stack

- React 19 + TypeScript
- Vite 6
- Tailwind CSS 4

## Run Locally

**Prerequisites:** Node.js

1. Install dependencies:
   ```
   npm install
   ```
2. Run the app:
   ```
   npm run dev
   ```

## Build

```
npm run build
```

## Deploy

Configured for [Railway](https://railway.app). Connect this repo in Railway and it will build and deploy automatically.
