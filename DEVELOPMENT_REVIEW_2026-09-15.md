# Development and Architecture Review — 2026-09-15

## Baseline and verdict

**Source HEAD:** `21c575206091cfb0d7117f1629a58f625d9909a9` on `main`, equal to fetched `origin/main` (`0` ahead, `0` behind). No commit follows the 2026-09-12 documentation review. Fresh tree comparison confirms no executable delta: `bot/` = `cfe5e308bbc01fc5b55329bc4378ac449720a70d` and `src/` = `eac7280ff4119b667d85fa2be66d3062fc35de58`, both identical to `21c5752^`.

**Verdict: not ready for unattended trading or meaningful capital.** Fresh source inspection reconfirms the open P0 paths below. No bot process, credential, live exchange request, order, cancellation or trade was used.

## Current critical gaps

1. **Unauthenticated write boundary and no independent enable gate.** Wildcard CORS remains at `bot/server.js:34-36`; start, stop, config and rebalance accept no control credential at `bot/server.js:56-105`. Start checks only exchange credentials (`bot/index.js:275-278`), so configured credentials implicitly enable writes.
2. **Cancellation can release live exposure.** `cancelAll()` suppresses individual failures and returns only successes (`bot/dextrade.js:184-193`). Rebalance and stop ignore result cardinality and mark every requested ID cancelled (`bot/index.js:161-166,301-310`).
3. **Uncertain reads can authorize writes and fabricate fills.** Order-book and balance failures are absorbed (`bot/index.js:51-90`). Open-order failure only returns from reconciliation, after which the tick continues (`bot/index.js:94-103,246-257`). One unpaginated open-order absence becomes a full fill at submitted floating-point values (`bot/index.js:105-125`; `bot/dextrade.js:159-179`).
4. **Placement is neither intent-first nor ambiguity-safe.** Requests use process-local millisecond IDs (`bot/dextrade.js:114-176`). The controller records a missing exchange ID as `"undefined"`; timeout/malformed outcomes have no durable hold, and the placement batch continues (`bot/index.js:208-230`).
5. **Restart and concurrency remain unsafe.** Order/PnL state is only in memory (`bot/state.js:3-48`), startup performs no adoption scan, and `tickInProgress` protects scheduled ticks but not HTTP stop/config/rebalance racing remote awaits (`bot/index.js:27-32,273-332`).

## Fresh non-trading evidence

| Gate | Result |
|---|---|
| Fetch and branch readback | **PASS** — HEAD = `origin/main` = `21c575206091cfb0d7117f1629a58f625d9909a9`; `0 0` |
| Runtime tree comparison | **PASS** — `bot/` and `src/` match the 2026-09-12 pre-publication trees |
| `npm run build` | **PASS** — Webpack `5.99.9`, 45.7 KiB production bundle, compiled in 796 ms; stale Browserslist-data warning |
| `node --check bot/*.js bot/strategies/*.js` | **PASS** — 9 files |
| `npm audit --omit=dev` | **FAIL** — 2 vulnerable entries: 1 high, 1 low |
| `(cd bot && npm audit --omit=dev)` | **FAIL** — 6 vulnerable entries: 3 high, 2 moderate, 1 low |
| Checked-in tests / CI | **ABSENT** — neither package defines `test`; no `*.test.js`, `*.spec.js`, or `.github` workflow is tracked |
| Optional injected HTTP/cancellation probes | **BLOCKED** by execution approval; not rerouted and not counted as evidence |
| Live exchange semantics | **NOT RUN** by safety scope |

Logs from this review were retained outside the repository at `/tmp/amm-sept15-{build,syntax,root-audit,bot-audit}.log`.

## Decision and next batch

Keep writes disabled and credentials absent. Execute only the containment/testability batch now specified at the top of [`DEVELOPMENT_PLAN.md`](DEVELOPMENT_PLAN.md): extract an injected side-effect-free controller, add default-deny enable/auth/origin/cap policy, and install one network-denying root test/CI gate. Its exit requires default configuration to produce zero controller/exchange calls on every mutation route. Then repair cancellation/read ambiguity under failing regression tests before adding any placement capability. The clean dependency direction is refreshed in [`ARCHITECTURE.md`](ARCHITECTURE.md); the durable-journal versus proven exchange-idempotency decision remains unresolved.