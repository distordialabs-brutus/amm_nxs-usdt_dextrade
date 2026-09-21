# Development and Architecture Review — 2026-09-21

## Baseline, delta and verdict

**Reviewed source HEAD:** `8880f1e8a3a1800bcf547494bfac65d084b15ba9` on `main`, equal to freshly fetched `origin/main` (`0` ahead, `0` behind) before this documentation update.

The only commit after the 2026-09-17 reviewed HEAD (`7017929848d4ddef4158d868e2cd4433db09416f`) is the prior review publication. Its path delta is `ARCHITECTURE.md`, `DEVELOPMENT_PLAN.md` and new `DEVELOPMENT_REVIEW_2026-09-17.md`. Runtime trees remain `bot/` `cfe5e308bbc01fc5b55329bc4378ac449720a70d` and `src/` `eac7280ff4119b667d85fa2be66d3062fc35de58`. No executable, manifest or lockfile changed.

**Verdict: unchanged — not ready for unattended trading or meaningful capital.** No runtime regression was introduced because runtime is source-identical, but no release blocker exited. This review made no dex-trade request, loaded no real credential, and created/cancelled no order. Production modules were exercised only with loopback/in-memory process, HTTP and exchange boundaries.

## Fresh executed evidence

The targeted probe used the real `bot/server.js`, `bot/index.js`, `bot/state.js`, strategies and controller call paths. External exchange and process boundaries were replaced before module load; credentials were synthetic; child processes used a temporary HOME; the HTTP probe listened only on ephemeral `127.0.0.1`. The probe is temporary review evidence, not a checked-in or CI-collected test.

| Gate / probe | Fresh result |
|---|---|
| Fetch and identity | **PASS** — pre-edit HEAD = fetched `origin/main` = `8880f1e8a3a1800bcf547494bfac65d084b15ba9`; `0 0` ahead/behind |
| Runtime delta | **PASS** — only prior review docs changed; `bot/` and `src/` tree IDs unchanged |
| Production build | **PASS** — Webpack `5.99.9`, 45.7 KiB bundle, compiled in 737 ms; stale Browserslist-data warning |
| Bot syntax | **PASS** — `node --check` over all 9 tracked bot/strategy JavaScript files |
| Direct production dependency trees | **PASS** — root: `nexus-module@1.1.11`, `react-redux@8.1.2`, `redux@4.1.2`; bot: `axios@1.13.5`, `cors@2.8.6`, `dotenv@16.6.1`, `express@4.22.1` |
| Root/bot collected tests | **ABSENT / FAIL** — both `npm test` commands return `Missing script: "test"` |
| Tracked CI | **ABSENT** — `.github` does not exist; no YAML workflow content was found |
| Online root production audit | **FAIL** — 2 entries: 1 high (`browserslist`), 1 low (`@babel/core`) |
| Online bot production audit | **FAIL** — 6 entries: 3 high, 2 moderate, 1 low; direct `axios@1.13.5` and transitive `body-parser`, `follow-redirects`, `form-data`, `path-to-regexp`, `qs` entries |
| Loopback control boundary | **FAIL safety gate** — attacker-origin start/config/rebalance/stop all returned 200, each returned `Access-Control-Allow-Origin: *`, and all 4 invoked the fake controller |
| Strategy schema/cap boundary | **FAIL safety gate** — `numGrids: 1000000` and `unknownExposureKey: 7` were forwarded unchanged |
| Import safety | **FAIL safety gate** — importing `bot/index.js` attempted 1 listener, 1 interval, SIGINT/SIGTERM handlers, and initial ticker/book/balance reads |
| Read uncertainty | **FAIL safety gate** — 3 failed balance reads were absorbed and 2 orders were placed from cached balances |
| Terminal evidence | **FAIL safety gate** — an empty open-order scan changed 2 orders to `filled` and fabricated 10 NXS buy, 10 NXS sell and 2 USDT realized PnL from submitted values |
| Placement attribution | **FAIL safety gate** — 2 `{}` placement responses produced one `managedOrders.undefined` row retaining only the last side |
| Cancellation cardinality | **FAIL safety gate** — 1 returned success for IDs A/B/C led all 3 local rows to `cancelled` |
| Stop/placement race | **FAIL safety gate** — stop returned `stopped` with 0 cancellation calls while placement was pending; after acceptance one order was locally `open`, and `start()` installed a post-stop interval |
| Live exchange semantics | **NOT RUN** by safety scope |

## Findings in financial-risk order

### 1. P0 — stop can report terminal while accepted exposure escapes cancellation

`start()` sets `running`, awaits the first `tick()`, and only then installs its interval (`bot/index.js:275-292`). `stop()` independently sets `running=false`, clears the current timer, snapshots only already-recorded managed IDs, optionally cancels that snapshot, and reports stopped (`bot/index.js:295-317`). Placement records its exchange ID only after `createLimitOrder()` resolves (`bot/index.js:208-230`).

The barrier probe paused the real controller in that await. Stop found no ID and returned `stopped`; resolving the fake accepted response then created `managedOrders.accepted-after-stop-scan` with status `open`. When the original start call resumed, it installed another interval. This is direct executable evidence of an orphan-order/false-stop path, not only a source hypothesis.

### 2. P0 — unauthenticated loopback mutation remains executable and unbounded

`bot/server.js:34-36` enables wildcard CORS. Routes at `bot/server.js:56-105` have no capability check and validate strategy values only as finite numbers. The real server accepted all four attacker-origin mutation routes and passed million-grid plus unknown-key configuration unchanged. Loopback binding lowers network exposure but does not authenticate browser or local-process callers; browser private-network behavior was not tested and is not a server control.

### 3. P0 — read failures and negative evidence still authorize writes/accounting

`fetchBalances()` absorbs all errors (`bot/index.js:75-90`), so both tick and rebalance can continue with stale snapshots. `reconcileOrders()` treats absence from one open-order result as a full fill and books submitted order values with no fill/fee evidence (`bot/index.js:94-125`). `getOpenOrders()` supplies no pair, page/cursor or completeness evidence (`bot/dextrade.js:156-165`), and `getOrderHistory()` is unused. Fresh execution reproduced cached-balance placement and fabricated terminal/PnL state.

### 4. P0 — write attribution and cancellation finality fail closed nowhere

Placement converts a missing canonical identity to string `undefined`, continues the batch after every caught submission error, and keeps no durable unknown-outcome reservation (`bot/index.js:208-230`). Millisecond references are generated immediately before calls (`bot/dextrade.js:114-176`) and are neither durable nor serialized. `cancelAll()` suppresses individual errors and shrinks result cardinality (`bot/dextrade.js:184-193`); stop/rebalance ignore those results and mark every requested ID cancelled (`bot/index.js:161-166,301-310`). Fresh execution reproduced both failure modes.

### 5. P0/P1 — restart recovery, exact money and reconciliation remain absent

`bot/state.js` is process-local. Startup has no complete adoption/recovery scan. Prices, volumes, minimums, balances and PnL use binary floating point and hard-coded four-decimal/5-USDT policy instead of exchange symbol and fee metadata. Open and unknown exposure are not durable liabilities. There is no reconciliation health that fails on zero checked entities, malformed rows or incomplete history.

### 6. Test, dependency and operational gates remain inadequate

Importing the composition module starts process effects (`bot/index.js:334-372`), both packages have no test command, and no CI workflow is tracked. Current online audits remain red. Dependency upgrades are available, but applying them before behavior and Nexus Wallet compatibility tests would exchange known advisory exposure for unmeasured trading/runtime risk.

## Positive controls actually verified

- The server remains bound to loopback in the composition path.
- Scheduled ticks retain the existing `tickInProgress` overlap guard; this does not serialize HTTP lifecycle commands.
- Exchange HTTP remains centralized in `bot/dextrade.js`; strategies remain I/O-free target generators.
- The production frontend build and all tracked bot syntax checks pass.
- The review worktree was clean before documentation edits, and runtime source was identical to the prior review baseline.

## Precise repair ordering and exit tests

1. **Contain and make import-safe.** Extract controller/config/composition, add default-false `TRADING_ENABLED`, and install one root network-denying `node:test` gate plus CI. **Exit:** imports produce zero listener/timer/signal/credential/network effects; missing enable/auth/origin/caps causes zero private exchange calls; CI runs the same collected command on every push.
2. **Serialize lifecycle and authenticate control.** Put start/stop/config/tick/rebalance/shutdown behind one queue; close admission before stop joins/resolves active work. Require exact origin, non-bundled capability, strict schemas and exposure caps. **Exit:** deterministic barriers at every exchange await cannot produce an open order or timer after terminal stop; all attacker-origin, wrong/missing capability, unknown/fractional/million-grid and over-cap requests return 4xx/503 with zero controller/state/exchange calls.
3. **Make reads complete and uncertainty held.** Validate pair/schema/freshness/pagination for book, balances, open and closed/fill reads; remove cached-balance and last-price authorization. **Exit:** transport, malformed, stale, crossed, wrong-pair and incomplete/page-budget cases produce a visible hold and exactly zero cancel/place/PnL effects; empty open orders never infer a terminal state.
4. **Persist attributable writes and exact cancellation results.** Choose a durable journal or prove target exchange idempotent reference/direct lookup; freeze quantized terms before submission and reserve unresolved exposure. **Exit:** fault injection at every intent/submit/identity/finalize/restart boundary proves one attributable action; `{}`/timeout aborts after one call with no `undefined` key; three cancellation IDs always yield three typed results and only positive evidence finalizes each.
5. **Adopt on restart and rebuild accounting from exact fills.** Startup remains held through complete adoption; use explicit decimal/integer policy and exact execution/fee evidence. **Exit:** open/unknown/partial/cancelled/filled restart fixtures neither duplicate nor fabricate; unequal-decimal, multi-fill, fee and rounding fixtures assert exact values; incomplete or zero-entity reconciliation is unhealthy.
6. **Validate external semantics, then remediate dependencies.** Use only capped disposable sandbox/test-account exposure after steps 1-5. **Exit:** target evidence proves pagination, precision/minimums, fees, cancellation finality, idempotency/direct lookup and timeout-after-acceptance; both production audits meet policy with build, tests and Nexus Wallet smoke green.

Publication SHA, sanitized origin and exact-head CI readback are external publication evidence and are reported after this document is committed and pushed; they cannot truthfully be embedded in the commit that creates them.