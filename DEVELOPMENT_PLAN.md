# NXS/USDT AMM Development Plan

This plan turns [`ARCHITECTURE.md`](ARCHITECTURE.md) into ordered, reviewable gates. Every implementation batch must keep the frontend build and bot syntax green, add collected regression coverage, deny unexpected network access, and preserve a default-disabled trading boundary.

## Current status — independent review 2026-09-21

Reviewed source HEAD is `8880f1e8a3a1800bcf547494bfac65d084b15ba9`, equal to freshly fetched `origin/main` before review publication. Since the 2026-09-17 reviewed HEAD (`7017929848d4ddef4158d868e2cd4433db09416f`), only that review's three documentation files changed. Runtime trees remain `bot/` `cfe5e308bbc01fc5b55329bc4378ac449720a70d` and `src/` `eac7280ff4119b667d85fa2be66d3062fc35de58`; no manifest or lockfile changed.

Fresh execution passed the production build, all nine tracked bot JavaScript syntax checks, and both direct production dependency-tree checks. Both packages still lack a test script and no `.github` workflow exists. Online production audits still fail: root `2` entries (`1` high, `1` low), bot `6` (`3` high, `2` moderate, `1` low). Isolated production-path probes reconfirmed unauthenticated attacker-origin mutation, stale-read placement, absence-based fabricated fills/PnL, missing-ID collapse and false cancellation finalization. A new deterministic barrier execution proved the documented stop race: an accepted order became locally open after stop returned, and the still-running `start()` call installed a post-stop interval. Every P0 remains open. See [`DEVELOPMENT_REVIEW_2026-09-21.md`](DEVELOPMENT_REVIEW_2026-09-21.md).

## Mandatory repair order

Do not reorder these batches to add trading capability before containment and its collected gate. Do not use live exchange credentials in Batches A-D.

### Batch A — import-safe seam, one gate, default deny (P0 containment)

**Change:** reduce `bot/index.js` to composition; add injected `bot/controller.js` and `bot/config.js`; make server construction import-safe; add a root `test` command, `test/network-deny.js`, `test/import-safety.test.js`, and `.github/workflows/ci.yml`. Parse an independent `TRADING_ENABLED` switch as false unless explicitly enabled. Missing control policy or exposure policy keeps mutation disabled before controller construction can reach private exchange methods.

**Exit command:** one documented root `npm test` command runs every collected test from a clean install with real DNS/socket/axios access denied, then CI runs the same command on every push.

**Exit assertions:** importing controller, server, config and domain modules creates zero listeners, timers, signal handlers, credential reads or network calls; default configuration permits read-only liveness/status only; start/config/rebalance cause zero private calls when enablement, auth/origin or caps are missing. Re-run the 2026-09-21 import probe and require `0` for listener, interval, signal and exchange-read counts.

### Batch B — authenticated control schemas and one lifecycle queue (P0 containment)

**Change:** enforce exact configured origin plus a non-bundled control capability on every mutation route; install server-owned strategy schemas and hard per-order, batch, inventory and unresolved-outcome caps. Route scheduled tick, start, stop, config, forced rebalance and shutdown through one controller queue. Stop closes admission first, joins/resolves the active transition, reconciles attributable exposure, and prevents any later interval installation.

**Exit tests:** `test/server.test.js` rejects disabled, missing/wrong capability, attacker origin, unknown field, fractional count, `numGrids=1000000`, and over-cap notional with 4xx/503 and exactly zero controller/state/exchange calls. `test/controller-races.test.js` places deterministic barriers before and after every exchange await; a stop during accepted placement must not return terminal until the order is either positively cancelled/closed or visibly held, no order may appear open after a terminal stop, and no timer/write may start after admission closes.

### Batch C — complete typed reads before any financial write (P0)

**Change:** return typed evidence for ticker/book, balances, open orders and fill/closed history: pair, fetched-at/freshness, validated schema, canonical IDs, cursor/range and explicit completeness. Remove last-trade fallback for quoting and cached-balance authorization. Open-order absence never implies fill or cancellation.

**Exit tests:** `test/read-evidence.test.js` injects transport failure, malformed envelope/row, wrong pair, stale/crossed book, missing required balance, non-monotonic/incomplete pagination and page-budget exhaustion through the actual controller. Every case records an operator-visible hold and causes zero cancellation, placement or PnL mutation. The captured empty-open-list case must leave both orders unresolved and PnL exactly unchanged; failed balance refreshes must permit zero placements despite a populated cached snapshot.

### Batch D — attributable writes, exact cancellation cardinality, durable ambiguity (P0)

**Change:** choose the durable-intent journal path or first prove target dex-trade idempotent client references plus direct lookup. Freeze pair, side, quantized price/volume, notional and unique request reference before submission. Persist/reserve intent before calling the exchange. Missing ID, malformed success, timeout or interruption becomes `outcome_unknown`, aborts the batch and blocks replacement. Cancellation returns one typed result per requested ID: `confirmed_cancelled`, `still_open` or `outcome_unknown`.

**Exit tests:** `test/order-lifecycle.test.js` fault-injects before intent, after intent/before call, after remote acceptance/before parse, after response/before identity persistence, after persistence/before finalization, restart and duplicate invocation. Every run proves one attributable remote action, retained unresolved exposure and no later-batch/replacement call. `test/cancellation.test.js` requires result cardinality equal to requested IDs; one success for three IDs may finalize only that one. A `{}` placement response must make exactly one call, create no `undefined` key and enter a durable held state.

### Batch E — startup adoption, exact fills and accounting (P0/P1)

**Change:** startup begins held and completely enumerates attributable open, closed, partial and unknown orders before first placement. Consume positive fill evidence with exact executed base/quote quantity, price, fee amount/asset and identity. Use explicit decimal or integer-scaled values, target symbol metadata and quantize before minimum/balance/cap checks.

**Exit tests:** crash/restart fixtures cover live owned, unknown/unowned, partial, cancelled and filled orders without duplicate submission or fabricated terminal state. Unequal-decimal, rounding-boundary, partial-fill, multi-fill and fee-currency fixtures assert exact expected balances, reservations and realized/unrealized PnL. A deliberate duplicate/overpayment produces a positive reconciliation discrepancy; zero checked entities or incomplete history is unhealthy, never green.

### Batch F — target exchange semantics, dependencies and operations (P1/P2)

**Change:** only after A-E pass, use a capped disposable dex-trade sandbox/test account to establish pagination, pair filtering, symbol precision/minimums, fee fields, cancellation finality, idempotency/direct lookup and timeout-after-acceptance behavior. Upgrade dependencies only under the complete gate; document credentials, capabilities, held-state recovery, safe stop and incident response.

**Exit:** recorded capped test-account cases prove each external semantic that mocks cannot. `npm audit --omit=dev` meets the documented policy in both packages; build, collected tests and Nexus Wallet installation/polling/control smoke remain green. No production-readiness claim is allowed if the exchange lacks a proven recovery mechanism for ambiguous submissions.

## Release-gate status

| Priority | Gate | Executable exit criterion | Status |
|---|---|---|---|
| P0 | Default-deny containment | Missing enable/auth/origin/cap configuration permits zero private exchange writes | **Not implemented** |
| P0 | Test/CI contract | One network-denying collected command covers controller, adapter, strategy and HTTP behavior on every push | **Not implemented** |
| P0 | Control authorization/input limits | Untrusted origin/credential and out-of-schema/over-cap requests are rejected with zero effects | **Not implemented** |
| P0 | Serialized lifecycle/safe stop | Await barriers cannot produce an open order or interval after terminal stop | **Not implemented; failure executed 2026-09-21** |
| P0 | Fresh complete read evidence | Failed/malformed/stale/incomplete reads cause a visible hold and zero writes/PnL | **Not implemented** |
| P0 | Durable attributable writes | Ambiguous writes survive restart, retain exposure and prohibit later-batch/retry/replacement | **Not implemented** |
| P0 | Exact cancellation cardinality | Every requested ID has a typed result; only positive evidence finalizes it | **Not implemented** |
| P0 | Restart reconciliation | Startup adopts or holds all attributable/unknown exposure before placement | **Not implemented** |
| P1 | Exact fill/fee accounting | Fill-level quantities, prices, decimals and fees produce deterministic exact results | **Not implemented** |
| P1 | Live exchange semantics | Capped sandbox/test-account cases prove pagination, finality and ambiguity recovery | **Not implemented** |
| P2 | Dependency remediation | Audits meet policy with build, tests and wallet compatibility green | **Deferred / compatibility-gated** |
| P2 | Operational runbook | Credentials, control capability, holds, stop and incident response are exercised | **Partial** |

**Release decision:** do not use meaningful capital or unattended operation until every P0 item passes, P1 money behavior receives independent review, and capped live-boundary evidence is complete.