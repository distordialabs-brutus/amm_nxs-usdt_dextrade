# Development and Architecture Review — 2026-09-08

**Reviewed base/HEAD:** `18e593c37f451d3c0cbab75aec19726aead2519e` (`main`).

**Prior review base:** `41c940fbbcafaf03c04fc19b22929630c220e73f`.
**Comparison:** `41c940f..18e593c`; the sole commit is the 2026-09-07 documentation publication (`ARCHITECTURE.md`, `DEVELOPMENT_PLAN.md`, `DEVELOPMENT_REVIEW_2026-09-07.md`). No application source changed. After fetch, local `main` equals `origin/main` with `0/0` divergence; the initial worktree was clean.

## Verdict

**Not ready for unattended trading or meaningful capital.** Prior cancellation, reconciliation, balance, placement-identity, restart, PnL and concurrency findings remain unfixed. This review adds current-tree control-plane authorization/input-validation and failed-market-evidence gates. These are newly identified historical defects, not regressions introduced by the documentation-only commit.

## Critical fund-loss or unauthorized-trading paths

1. **P0 — browser origins can control a credentialed local trading bot.** `bot/server.js:31-36` enables `cors({ origin: '*' })`; mutating routes at `:56-105` require no control credential. Loopback binding is not server authorization; the server grants cross-origin access, although actual browser private-network/mixed-content restrictions were not tested. An isolated probe confirmed that `https://untrusted.example` receives `Access-Control-Allow-Origin: *` and a POST reaches the mocked start controller. Require authenticated mutation requests and an exact configured origin policy; keep mutations disabled when configuration is absent.
2. **P0 — cancellation and stop fabricate terminal state.** `bot/dextrade.js:184-193` drops failed cancellation outcomes. Rebalance (`bot/index.js:161-166`) and stop (`:301-310`) mark every requested ID cancelled anyway. Existing live orders can remain while replacements are placed or stop reports completion.
3. **P0 — failed/malformed reads do not reliably stop writes.** Open-order transport failure returns from reconciliation (`bot/index.js:94-103`) but not from the tick, so rebalance can continue. A malformed non-array response without `list` becomes `[]` (`:98-110`) and missing orders become full fills. Balance failures are absorbed (`:75-90`) and cached balances are reused (`:169-172`). Order-book failure is absorbed and last trade becomes the quote center (`:51-71`), risking stale or crossing orders.
4. **P0 — placement acceptance is not attributable.** `bot/index.js:208-230` stores a missing ID under the string `undefined`; a timeout has no explicit intent/outcome-unknown record. Blind later replacement can duplicate remote exposure.
5. **P0 — stop can race an in-flight placement.** The tick guard serializes ticks, not HTTP controller commands. Stop can capture its cancellation set while create-order is awaiting a response; the accepted order can be recorded after that set was built and escape cancellation.

## High money-contract and reconciliation defects

- Missing open orders are booked at full submitted quantity/price with no closed-order or fill lookup (`bot/index.js:105-125`). Partial fills and cancellations therefore corrupt inventory and PnL; fees remain unused.
- Restart loses managed-order/PnL state and immediately permits a first tick; there is no exchange adoption or reconciliation hold.
- Server validation checks only finite numeric shape (`bot/server.js:16-29`), not strategy-owned keys, schema min/max, integer counts or resource ceilings. The isolated probe confirmed `numGrids=1000000` reaches the controller. Frontend HTML constraints are not a server control; oversized loops can deny service and arbitrary valid-shaped values can radically alter exposure.
- Shared rate-limit timestamps are not atomic (`bot/dextrade.js:11-22`), so control/tick concurrency can violate request spacing.

## Test, live-evidence and dependency gaps

- Neither package defines a test script, and no tracked CI workflow exists. The isolated review probe is evidence for this review only, not a repository regression gate.
- No live exchange call, API credential, trading process or financial mutation was used. Mocked probes do **not** prove dex-trade response schemas, pagination completeness, canonical IDs, fees, partial fills, cancellation finality or timeout-after-acceptance behavior.
- `npm audit --omit=dev` failed for the root package with **2 production findings** (1 high, 1 low: Browserslist and Babel advisories) and for `bot/` with **6 production findings** (3 high, 2 moderate, 1 low: Axios plus transitive body-parser, follow-redirects, form-data, path-to-regexp and qs advisories). Fixes are reported available, but upgrades were not applied without tests and wallet compatibility evidence.

## Positive controls and executed gates

| Gate | Result |
|---|---|
| `git fetch --prune origin`; local/remote comparison | **PASS** — both `18e593c37f451d3c0cbab75aec19726aead2519e`, divergence `0 0` |
| `npm run build` | **PASS** — webpack 5.99.9, 45.7 KiB production bundle; stale Browserslist-data warning |
| `node --check` over 9 bot/strategy files | **PASS** |
| Isolated strategy fixtures | **PASS** — all three defaults returned finite positive orders |
| Isolated signing fixture | **PASS** — recursive sorted-value SHA-256 behavior matched |
| Isolated Express probe with mocked controller | **PASS as a probe** — health/start paths executed with no exchange call; it also reproduced wildcard CORS and missing schema bounds |
| `git diff --check` before review edits | **PASS** |
| Root and bot production dependency audits | **FAIL** — 2 and 6 findings respectively |
| Repository tests / CI | **ABSENT** — no test script or workflow |

Implemented controls observed in source are the loopback bind, non-overlapping scheduled ticks, centralized exchange adapter, sequential cancellation loop, 10-second HTTP timeouts, four-decimal order formatting, minimum-order and local available-balance checks, deep-copy status snapshots and pure default strategy generation. These do not close the evidence and concurrency defects above.

## Repair order and executable exit criteria

1. Extract controller construction from process startup and add a network-isolated `node:test`/CI gate. Imports must open no listener and make no exchange call.
2. Add fail-closed control authentication, exact origin policy and server-owned strategy validation. Untrusted origin/credential, unknown/out-of-range params, fractional counts and `numGrids=1000000` must return 4xx with zero controller/exchange calls.
3. Add typed, schema-valid, complete and fresh read evidence. Inject order-book, balance and order-enumeration transport/schema/pagination failures and assert operator-visible hold, zero placements/cancellations and zero PnL changes.
4. Add per-order cancellation outcomes and intent-first attributable placement. Inject partial cancellation, missing ID and accepted-but-timed-out responses; assert at most one remote action and no replacement until positive evidence resolves exposure.
5. Serialize tick/start/stop/config/rebalance transitions and request scheduling. Barrier tests at each await must prove no post-stop orphan order or rate-limit race.
6. Add startup adoption/hold and fill-level quantity/price/fee reconciliation. Fixture tests must cover open, cancelled, partial and full fills; capped sandbox/test-account tests must then establish real pagination and finality semantics.

The authoritative sequence and release gate are in [`DEVELOPMENT_PLAN.md`](DEVELOPMENT_PLAN.md); the updated system boundary and invariants are in [`ARCHITECTURE.md`](ARCHITECTURE.md).
