# Truss Analyzer — Release and deployment specification

**Status:** Draft for review  
**Scope:** Build, hosting, configuration, release verification, and rollback. Describes the current Railway setup and proposes a repeatable checklist.

## 1. Deployment shape

Truss Analyzer is a static single-page app. Vite builds `dist/`; there is no application server, database, or server-side secret in this repository. Hanger lookups are browser-to-SST calls using a user-supplied token in `localStorage`.

| Item | Current value |
| --- | --- |
| Node requirement | `>=20` (`package.json` engines). |
| Dev | `npm run dev` → Vite on port 3000, host `0.0.0.0`, with HMR unless `DISABLE_HMR=true`. |
| Build | `npm run build` → `vite build`, output `dist/`. |
| Preview | `npm run preview`; `npm start` → `vite preview --host 0.0.0.0 --port ${PORT:-3000}`. |
| Type check | `npm run lint` → `tsc --noEmit`; it does not emit or run tests. |
| Hosting | Railway using Nixpacks. `railway.json` builds with `npm install && npm run build` and starts with `npm run start`. |
| Restart policy | Restart on failure, up to 10 retries. |
| Preview hosts | Vite preview allows all hosts (`allowedHosts: true`). |
| Ignored artifacts | `node_modules/`, `build/`, `dist/`, `coverage/`, logs, and `.env*` (except `.env.example`). |

`vite.config.ts` aliases `@` to the repository root. A deployed SPA needs host fallback to `index.html` for client-side routes if routes are added; the current app uses tabs and state, not routes.

## 2. Configuration and secrets

- No build-time environment variables are required by the application code. `PORT` is supplied by Railway and used by `npm start`.
- The SST bearer token is entered in the UI and stored in browser `localStorage`. It must not be placed in Railway variables, source files, logs, or built assets.
- `.env*` files are ignored except `.env.example`. If proxy configuration or a public base URL is introduced later, document it here and add a non-secret example.
- The SST endpoint and enums are hard-coded in `src/lib/sst-types.ts`; changing them is a code release, not a configuration change.

## 3. Release checklist

### Pre-release

- [ ] Work is based on a known-good main branch with no unrelated changes.
- [ ] `npm install` from a clean lockfile succeeds on Node 20 or newer.
- [ ] `npm run lint` and `npm run build` pass; `dist/` is generated without warnings that matter.
- [ ] Testing and QA checklist in [testing-qa-spec.md](testing-qa-spec.md) passes.
- [ ] Accessibility checklist in [accessibility-spec.md](accessibility-spec.md) passes for changed screens.
- [ ] Semantic parser and workbook baselines are unchanged, or diffs have a domain approval and a recorded reason.
- [ ] No token, credential, raw project text, or debug logging of sensitive payloads is present in the built app.

### Deploy

1. Merge the reviewed change to the branch Railway deploys.
2. Confirm Railway uses the intended commit/branch and that the Nixpacks build runs `npm install && npm run build`.
3. Wait for the health of the deployment and confirm `npm run start` is serving on the provided `PORT`.
4. Open the deployed URL and run the smoke test below.

### Smoke test

- App loads with the empty state or sample only when appropriate; layout is usable at desktop and mobile widths.
- Import a small real IFC/TRE set; verify girder and connection counts against the baseline for that fixture.
- Open a carried connection and verify diagrams and reactions populate.
- With a fresh valid token, run one single lookup and one short batch; confirm results, no-match, auth-failure, and network-failure states can be distinguished.
- Import an inventory file, verify ranking, and download the workbook. Confirm both sheets, headers, and row counts.
- Confirm no sensitive values appear in the browser console during normal use.

## 4. Verification and monitoring

- There is no automated health-check or telemetry endpoint in this repository. Railway restart policy covers process failure, not a silently broken static build or a changed SST contract.
- After each deploy, verify the served asset set corresponds to the intended commit, and repeat the smoke test if the SST integration, parser, or packaged sample changed.
- Track SST failures by category from user reports: token expiry, CORS/network, provider schema or enum change. A sudden rise in 401s may mean the documented token-refresh flow is being hit; it is not a code regression by itself.
- If the provider changes the endpoint or response shape, treat it as an external breaking change: update the contract in [sst-integration-spec.md](sst-integration-spec.md) and rerun the full QA checklist.

## 5. Rollback

- Rollback means redeploying the previous known-good Railway commit. All job data is client-side and session-scoped, so no database rollback is needed.
- Users with an in-flight browser session should reload after rollback to avoid mixing old and new bundle state.
- If a bad release generated incorrect schedules, the downloaded workbooks cannot be recalled. Communicate the affected fixture/workbook pattern and provide a corrected build rather than relying only on rollback.

## 6. Future considerations

- Add a repeatable CI job for lint, build, and automated tests before release.
- Pin or cache the Nixpacks build environment where possible to avoid Node-version drift.
- If CORS blocks browser-to-SST calls on the deployed domain, evaluate a server-side proxy as a separate architectural change; it would introduce secrets, new hosting responsibilities, and a new deployment checklist.
- Add an explicit app version and build identifier in the UI so deployed sessions and bug reports can be tied to a release.
