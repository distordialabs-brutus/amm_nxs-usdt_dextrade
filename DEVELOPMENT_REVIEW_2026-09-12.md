# Development and Architecture Review — 2026-09-12

## Baseline and delta

**Reviewed pre-publication HEAD:** `6c17f069dc36e8eedf28cd5d88ab1840b436e4cb` on `main`, equal to fetched `origin/main` with `0` ahead and `0` behind. The worktree was clean before this review.

The only delta from the 2026-09-10 review's pre-publication source HEAD `10db348c947afcb508ea533ddf246603ed56beff` is that review's documentation: `ARCHITECTURE.md`, `DEVELOPMENT_PLAN.md`, and `DEVELOPMENT_REVIEW_2026-09-10.md`. Runtime trees remain exactly `bot/` = `cfe5e308bbc01fc5b55329bc4378ac449720a70d` and `src/` = `eac7280ff4119b667d85fa2be66d3062fc35de58`.

No exchange credentials were used. No bot process, live exchange read, order, cancellation, trade, dependency mutation, Nexus node call, or runtime repair was performed.

## Verdict

**Not ready for unattended trading or meaningful capital.** There is no executable delta addressing the prior money-safety blockers. The application can build, but the order lifecycle remains unauthenticated, non-durable, ambiguity-unsafe, absence-based, and race-prone.

## Findings, in risk order

### Critical fund-loss and exposure paths

1. **P0 — unauthenticated mutation boundary.** Wildcard CORS is enabled in `bot/server.js:31-36`; start, stop, config, and forced rebalance are callable without a control credential at `bot/server.js:56-105`. A loopback bind is containment, not authorization.
2. **P0 — cancellation manufactures terminal state.** `cancelAll()` suppresses per-order failures and returns a shortened result list (`bot/dextrade.js:184-193`). Rebalance and stop ignore those outcomes and mark every requested ID cancelled (`bot/index.js:161-166,301-310`). This can release exposure that remains live at dex-trade.
3. **P0 — uncertain reads authorize writes and fabricate fills.** Order-book and balance failures are logged and absorbed (`bot/index.js:51-90`). Reconciliation returns locally on an open-order failure (`bot/index.js:94-103`), but the tick continues to balance refresh and rebalance (`bot/index.js:246-257`). Absence from one unpaginated open-order response is treated as a full fill at submitted price and quantity (`bot/index.js:105-125`).
4. **P0 — placement is not intent-first or ambiguity-safe.** Request IDs are process-local millisecond timestamps (`bot/dextrade.js:128-139`). A timeout or malformed success is caught and the placement loop continues (`bot/index.js:208-230`); a missing exchange ID is persisted under the string `"undefined"` (`bot/index.js:209-217`). No durable unknown-outcome liability prevents retry or replacement.
5. **P0 — restart and lifecycle concurrency can orphan orders.** Financial state is process-local (`bot/state.js:6-48`), startup performs no adoption scan, and `tickInProgress` serializes scheduled ticks only. HTTP stop/config/rebalance can race a placement already awaiting exchange acceptance (`bot/index.js:273-332`).

### High money-contract and reconciliation defects

- `getOpenOrders()` has no pair, cursor, page, or completeness contract (`bot/dextrade.js:156-165`), while `getOrderHistory()` is unused (`bot/dextrade.js:167-179`). Negative open-order evidence is therefore not authoritative.
- Four-decimal formatting and the 5 USDT minimum are hard-coded (`bot/index.js:17,175-190`; `bot/dextrade.js:128-136`) rather than derived from validated symbol metadata. Volume is not frozen after quantization before balance/minimum decisions.
- Realized PnL is based on submitted floating-point values, assumes a missing order filled in full, and records no exact fill identity, execution price, fee amount, or fee asset (`bot/index.js:107-125`; `bot/state.js:34-42`).
- Strategy input validation accepts any finite numeric value and unknown key (`bot/server.js:16-29`); it does not enforce strategy-owned schemas, integer count fields, order-count ceilings, batch notional, or account exposure limits.
- The shared rate-limit timestamp is not protected against concurrent callers (`bot/dextrade.js:11-22`).

### Verification and operational gaps

- Neither package declares a test command and no CI workflow is tracked. Build and parser checks do not exercise financial invariants.
- Fresh production audit commands still fail: root reports `2` vulnerabilities (`1` high, `1` low); bot reports `6` (`3` high, `2` moderate, `1` low).
- Live dex-trade pagination, client-reference/idempotency behavior, cancellation finality, fill/fee schema, timeout-after-acceptance, and rate-limit semantics remain unverified.

## Executed evidence

| Gate | 2026-09-12 result |
|---|---|
| `git fetch --prune origin`; branch/remote readback | **PASS** — local and fetched remote both `6c17f069dc36e8eedf28cd5d88ab1840b436e4cb`; `0 0`; clean start |
| Delta and runtime tree identity | **PASS** — documentation-only delta; `bot/` and `src/` tree IDs unchanged |
| `npm run build` | **PASS** — Webpack `5.99.9`, 45.7 KiB production bundle; stale Browserslist-data warning |
| `node --check bot/*.js bot/strategies/*.js` | **PASS** — 9 files |
| `npm audit --omit=dev` | **FAIL** — 2 vulnerabilities: 1 high, 1 low |
| `npm audit --omit=dev --prefix bot` | **FAIL** — 6 vulnerabilities: 3 high, 2 moderate, 1 low |
| `git diff --check` before review docs | **PASS** |
| Configured tests / checked-in CI | **ABSENT** |
| Live exchange behavior | **NOT RUN** by safety scope |

## Decision

Keep all write paths disabled by default and do not use meaningful capital. Execute the exits in [`DEVELOPMENT_PLAN_ADDENDUM_2026-09-12.md`](DEVELOPMENT_PLAN_ADDENDUM_2026-09-12.md). The target order/recovery boundary is specified in [`ARCHITECTURE_ADDENDUM_2026-09-12.md`](ARCHITECTURE_ADDENDUM_2026-09-12.md). Earlier dated reviews remain historical evidence.
