# Truss Analyzer — SST integration specification

**Status:** Draft for review  
**Scope:** Contract, security, and state behavior for hanger lookup and inventory. Complements the [technical specification](technical-spec.md); no behavioral change is implied.

## 1. External contract

- **Endpoint:** `POST https://api.strongtie.com/gws/hanger-selector/hangers`.
- **Origin:** Direct browser fetch from the SPA. There is no proxy, backend, or API key in this repository.
- **Source of contract:** Reverse-engineered from a browser DevTools capture; enum values and response fields live in `src/lib/sst-types.ts`. The provider may change the endpoint or schema without notice.
- **Authentication:** `Authorization: Bearer <token>` with `Content-Type: application/json` and `Accept: application/json`. The token is pasted by the user, stripped of a `Bearer ` prefix, and stored in `localStorage` under `sst_bearer_token`.
- **Token lifetime:** The client notes tokens expire after about one hour and shows a refresh guide on a 401. A saved token means only that a value exists, not that it is valid.
- **Cross-origin risk:** If the API rejects the app origin, the browser reports an opaque network error. The current client labels it as a likely CORS issue and mentions that a proxy may be needed. A proxy is **not** in scope without a separate decision.

## 2. Request contract

`buildSSTPayload(group, carried)` is the single default payload builder. Overrides in the selector replace style, fastener type, ANSI/TPI classification, download duration, and uplift duration immediately before submission; batch lookup uses the default payload.

| Payload area | Source / rule |
| --- | --- |
| Carrying member | Girder bottom-chord cross-section, species/ply from TRE, king-post dimensions from a vertical structural web near the connection; defaults when unavailable. |
| Carried member | Carried bottom-chord width and heel height at the bearing side, species/ply, and connection angles. |
| Loads | Rounded absolute values of the derived downward reaction and uplift. A negative domain uplift is sent as a positive magnitude. |
| Angles | Hanger angle converts to skew magnitude plus a left/right/none type; carried bottom-chord slope at the bearing converts to an SST up/down/none type. |
| Durations | Governing load-case factors map to SST download/uplift duration enums, with roof/wind defaults when factors are missing. |
| Constants | Truss material type, IRC 2018 default, flush-bottom truss installation, all styles/fasteners by default. |

### 2.1 Required guardrails

- Build requests from the active job only. Tag each request with a correlation ID and an effective-payload fingerprint so stale responses cannot update a later connection or job.
- Do not send original IFC/TRE text or the bearer token inside request bodies or diagnostics.
- Treat override changes as new inputs: display existing results as out-of-date rather than silently presenting them as current.
- Keep the default payload used by the batch path independent of temporary selector overrides.

## 3. Response contract

The response is normalized to `{ success, hangers, error?, raw? }`.

| Outcome | Detection | UI meaning |
| --- | --- | --- |
| Success with candidates | `status.error` false, `lstHangerOutput` populated | Show rankable candidates and inform the summary. |
| Success with no candidates | Same response, empty list | A valid result: **no matches**, not a transport failure. |
| API status error | `status.error` true | Show the provider's text where safe; preserve raw only for debugging, never in exports. |
| Missing token | No stored token | Direct user to Settings; do not submit. |
| 401 / 403 | HTTP status | Invalid/expired or unauthorized token; direct user to Settings and stop a running batch. |
| Other HTTP failure | Non-OK status | Show bounded status/body text; allow retry. |
| Network / CORS-style failure | Fetch throws `Failed to fetch` / `NetworkError` | Show a network-or-CORS message; allow retry. Do not claim a specific cause the browser cannot prove. |

## 4. State model

### 4.1 Single connection

States: `idle → (no token) → blocked`, `idle → loading → found | no match | failed`, and `found → stale` when effective inputs change.

- Current behavior auto-queries when the output panel mounts or effective payload changes. **Proposed behavior:** submit only from an explicit **Search SST** action; view entry and field edits never send requests.
- Keep filters, sorting, and stock-only filtering local to the results display. They do not alter the candidate set passed to the summary.
- A successful result populates the shared per-connection map. A failed query must not erase a previous successful result; mark the previous result stale or failed-attempt as appropriate.

### 4.2 Batch

- Queue eligible connections from the active job that have no usable result and positive downward reaction.
- Send requests sequentially with a delay between calls to reduce rate-limit risk. Use cancellation only between requests unless an abortable fetch is deliberately introduced.
- Track per-connection outcomes so **not queried**, **running**, **found**, **no match**, **failed**, and **not eligible** are distinguishable in the UI. A stored empty candidate list means a completed no-match query.
- On stop, keep completed results and allow a later run to retry only failed or unqueried connections.
- Abort or discard a batch when the active job is replaced. Authentication failure stops the batch; other failures do not necessarily stop it.

## 5. Inventory integration

- File input accepts `HangerInventoryXml`. Parse `InventoryList > Inventory` entries; require a correct root and at least one entry.
- Availability is computed from `IsUnavailable` after normalizing `MaterialName` (trim, uppercase, whitespace collapse). Any available entry for a model makes that model in stock. Duplicate entries and prices do not affect the boolean.
- Keep one inventory in shared app state; the selector and summary must observe the same import/removal. Import errors are visible and do not silently clear good prior data unless the user retries or removes explicitly.
- Ranking rule: prefer the lowest positive installed cost index among in-stock candidates; otherwise rank all candidates the same way. Unknown/zero indices sort after positive values; ties preserve returned order. Do not change this rule as part of the UI rework.
- The workbook shows availability as **Yes/No** for the selected candidate. It is not a stock quantity and does not reserve inventory.

## 6. Security and privacy

- The token remains a user-supplied client-side credential. Mask it in the UI; never display it in plain text after saving.
- Do not log the token, raw response bodies containing it, or full project-derived payloads during normal operation. Existing console logging of payload/response should be removed or reduced to non-sensitive metadata.
- Do not write the token into Excel, diagnostics exports, or test fixtures.
- Make credential expiry recoverable: any 401 directs the user to refresh the token, and a batch stops rather than retrying an invalid credential for every connection.
- Treat the provider endpoint and enum table as an external contract. If the app is deployed on a domain the API blocks, decide explicitly between a server-side proxy and a documented browser-only limitation.

## 7. Acceptance checks

- A stubbed endpoint verifies request shape for representative girder/connection types and all response outcomes in section 3.
- Requests fire only from explicit user actions in the proposed flow; no request on mount or override edit.
- A late response after changing connection or replacing a job never writes to the active state.
- Batch stop, retry, auth failure, empty result, and non-auth failure behave as specified.
- Inventory normalization, duplicate entries, invalid XML, and removal are covered; ranking matches the baseline.
- With no token, analyzer, diagrams, and workbook export still work; only hanger lookup is blocked.
