# Development and Architecture Review — 2026-09-09

**Reviewed HEAD:** `f4d68934a1974a71c712229d6015af6abcaa1fed` (`main`).

**Prior review's reviewed base:** `18e593c37f451d3c0cbab75aec19726aead2519e`.

**Comparison:** `18e593c..f4d6893`; the only commit is the 2026-09-08 documentation publication (`ARCHITECTURE.md`, `DEVELOPMENT_PLAN.md`, `DEVELOPMENT_REVIEW_2026-09-08.md`). No executable source or dependency manifest changed. After fetch, local `main` and `origin/main` are equal with `0/0` divergence. The initial worktree was clean.

**Runtime identity:** HEAD tree `97df9f5bfbf03eed13996b372f43051d827f13f0`; `bot/` tree `cfe5e308bbc01fc5b55329bc4378ac449720a70d`; `src/` tree `eac7280ff4119b667d85fa2be66d3062fc35de58`. The `bot/` and `src/` tree IDs are identical at `18e593c` and HEAD, and a path-limited diff across runtime/configuration files is empty. Initial Git index SHA-256 was `d4790a0c651acfce16a6df8d5d2ecf3b15292172294db561c49876eac8803f8b`.

## Verdict

**Not ready for unattended trading or meaningful capital.** There is no runtime delta to resolve the 2026-09-08 findings. The executable money boundary still permits unauthorized control, fabricated cancellation/fill state, writes after uncertain reads, duplicate exposure after ambiguous placement or restart, and races between stop and placement. This review sharpens two architecture gaps: the no-database rule cannot provide crash-safe intent by itself, and open-order enumeration has no completeness/pagination proof.

## Critical fund-loss or unauthorized-trading paths

1. **P0 — unauthenticated wildcard-CORS mutation remains reachable.** `bot/server.js:31-36,56-105` accepts start, stop, config and rebalance commands without a control credential and returns `Access-Control-Allow-Origin: *`. Loopback is not authorization. The 2026-09-08 isolated probe reproduced the route reachability; it was not re-run in this review.
2. **P0 — cancellation and stop still fabricate terminal state.** `bot/dextrade.js:184-193` discards failed per-order outcomes. Rebalance and stop then label every requested order cancelled (`bot/index.js:161-166,301-310`) without confirmed exchange finality. Live orders can survive while replacements are placed or shutdown exits successfully.
3. **P0 — failed, malformed or incomplete reads can authorize writes and fake fills.** Order-book failure is replaced with last trade (`bot/index.js:51-71`), balance failure is absorbed and can leave cached values (`:75-90,169-172`), and reconciliation failure returns only from the helper (`:94-103`) before the tick continues. `getOpenOrders()` has no pagination/completeness contract (`bot/dextrade.js:156-165`); omitted or malformed-ID records can therefore make locally managed orders appear filled (`bot/index.js:105-125`).
4. **P0 — placement is neither intent-first nor ambiguity-safe.** The only request reference is `String(Date.now())` created immediately before transport (`bot/dextrade.js:128-139`); it is not persisted, proven unique/idempotent, or linked to a recovery lookup. Missing IDs become the key `"undefined"` (`bot/index.js:208-218`). A timeout is logged and the batch continues (`:228-230`) without reserving the unknown exposure, so later orders and the next rebalance can compound a remotely accepted order.
5. **P0 — restart and stop can duplicate or orphan exposure.** State is process-local, startup immediately permits a first tick, and remote open orders are not adopted. The tick guard does not serialize HTTP stop/config/rebalance with a placement already awaiting a response. Stop can build its cancellation set before that response and miss the accepted order.

## High money-contract and reconciliation defects

- Missing open orders are booked at full submitted quantity and price; `getOrderHistory()` exists but has no caller. Partial fills, cancellations, execution price and fees are not reconciled.
- A durable pre-submit intent is absent. Under the current no-database constraint, crash-safe at-most-once placement requires a proven exchange idempotency/reference contract plus direct authoritative recovery; otherwise unattended recovery is architecturally unsupported. An in-memory intent alone is not an exit criterion.
- Price and volume precision are hard-coded to four decimals and the minimum notional to 5 USDT (`bot/index.js:17,175`; `bot/dextrade.js:133-135`) without consumed symbol metadata. Balance/minimum checks use raw strategy volume before the adapter rounds transmitted volume, so the checked and submitted amounts can differ.
- Server validation accepts finite numbers but does not enforce strategy ownership, schema bounds, integer counts, order-count ceilings or account/batch notional caps (`bot/server.js:16-29`).
- Shared rate-limit timestamps and millisecond request IDs are not serialized across concurrent callers (`bot/dextrade.js:11-22,114-176`).
- PnL remains floating-point, aggregate weighted-average submitted value and ignores exact fills and fee currency.

## Test, live-evidence and dependency gaps

- Neither package defines a test script and no `.github` workflow exists. There is no repository regression gate for the money state machine.
- No live exchange call, API credential, trading process or financial mutation was used. Local evidence does not prove dex-trade schemas, request-ID semantics, precision/minimum metadata, pagination, cancellation finality, fills, fees or timeout-after-acceptance recovery.
- Fresh `npm audit --omit=dev` failed: the root report contains **2 vulnerable packages** (1 high, 1 low), and the `bot/` report contains **6** (3 high, 2 moderate, 1 low). Fixes are reported available but were not applied without behavior and wallet compatibility gates.
- New isolated strategy/signature probe commands were explicitly approval-denied and were not retried or reformulated. Their 2026-09-08 passes remain historical evidence only.

## Positive controls and executed gates

| Gate | Result |
|---|---|
| `git fetch --prune origin`; local/remote comparison | **PASS** — both `f4d68934a1974a71c712229d6015af6abcaa1fed`, divergence `0 0` |
| Runtime delta/hash check | **PASS** — no runtime/configuration diff; `bot/` and `src/` tree IDs unchanged from `18e593c` |
| `npm run build` | **PASS** — webpack 5.99.9, 45.7 KiB production bundle; stale Browserslist-data warning |
| `node --check` over all 9 bot/strategy files | **PASS** |
| `git diff --check` before review edits | **PASS** |
| Root and bot production dependency audits | **FAIL** — 2 and 6 vulnerable packages respectively |
| Configured repository tests / CI | **ABSENT** |
| New isolated strategy/signature probes | **BLOCKED** by explicit approval denial; not retried |
| Live exchange behavior | **NOT RUN** by safety scope |

Positive controls observed, but not sufficient for release, remain loopback binding, non-overlapping scheduled ticks, a centralized exchange adapter, sequential cancellation attempts, 10-second HTTP timeouts, local minimum/balance checks, deep-copy status snapshots and pure strategy generation.

## Prioritized executable repair plan

1. **P0 containment now:** keep credentials removed or trading disabled outside a capped disposable test account. Add a default-deny trading enable gate; absent explicit enable/auth/origin/exposure configuration, mutation routes must make zero exchange calls.
2. **P0 deterministic test seam:** separate controller construction from listener/process startup and add one network-denying `node:test` command plus CI. Imports must not listen, install signal handlers, load real credentials or call axios.
3. **P0 control and admission policy:** authenticate every mutation, allow only the exact configured wallet origin, validate schema-owned parameters, and enforce integer/order-count/per-order/batch/account exposure caps. Rejection fixtures must prove zero state and adapter calls.
4. **P0 fail-closed reads:** add typed schema validation, freshness and complete pagination evidence for market, balances, open orders and fills. Inject transport/schema/page-budget failures and assert visible hold, zero cancellation/placement and zero PnL mutation.
5. **P0 attributable writes:** decide the durability architecture. Persist intent before submission, or prove the exchange request reference is unique, idempotent and directly recoverable across restart. Abort and hold the whole batch on an unknown outcome; confirm each cancellation before replacement. Fault injection at every await/crash boundary must prove one remote action, never zero-or-two by accident.
6. **P0 serialization and restart:** use one transition queue for tick/start/stop/config/rebalance and request scheduling. Startup must adopt attributable exposure or remain held; barriers must prove no post-stop orphan order and no duplicate request reference.
7. **P1 exact exchange contract and accounting:** consume symbol precision/minimum metadata, quantize before admission checks, and reconcile exact fill quantity, price and fees with integer/decimal-safe accounting. Fixture cases cover rounding boundaries, partial/multi-fill, cancellation and fees; capped sandbox/test-account cases establish the real external semantics.
8. **P2 dependency and operations:** upgrade only under the full gate, then exercise wallet integration, credential rotation, held-state recovery, stop and incident-response runbooks.

The authoritative sequence and release gate are in [`DEVELOPMENT_PLAN.md`](DEVELOPMENT_PLAN.md); the system boundary and invariants are in [`ARCHITECTURE.md`](ARCHITECTURE.md).
