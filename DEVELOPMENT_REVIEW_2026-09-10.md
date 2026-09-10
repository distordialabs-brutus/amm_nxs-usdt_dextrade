# Development and Architecture Review — 2026-09-10

## Baseline and delta

**Reviewed pre-publication HEAD:** `10db348c947afcb508ea533ddf246603ed56beff` (`main`), equal to `origin/main` at review start.

**Prior dated review's source HEAD:** `f4d68934a1974a71c712229d6015af6abcaa1fed`. The only intervening commit is the 2026-09-09 documentation publication. `git diff --name-status f4d6893..10db348` contains only `ARCHITECTURE.md`, `DEVELOPMENT_PLAN.md` and `DEVELOPMENT_REVIEW_2026-09-09.md`; a path-limited runtime/configuration diff is empty. The worktree was clean.

Runtime identity is unchanged: `bot/` tree `cfe5e308bbc01fc5b55329bc4378ac449720a70d`, `src/` tree `eac7280ff4119b667d85fa2be66d3062fc35de58`. Findings below are retained defects, not new regressions.

No live exchange call, credential use, bot process, order, cancellation, trade, dependency change or runtime repair was performed.

## Verdict

**Not ready for unattended trading or meaningful capital.** The unchanged money boundary still allows unauthenticated control, fabricated cancellation/fill state, writes after uncertain reads, duplicate exposure after ambiguous placement or restart, and stop/placement races. Buildability does not establish trading safety.

## Critical retained findings

1. **P0 — mutation routes are unauthenticated and wildcard-CORS.** `bot/server.js:31-36,56-105` exposes start, stop, config and rebalance without a control credential. Loopback is not authorization.
2. **P0 — cancellation and stop manufacture terminal state.** `bot/dextrade.js:184-193` drops failed per-order outcomes; `bot/index.js:161-166,301-310` then marks every requested order cancelled without exchange proof.
3. **P0 — uncertain/incomplete reads can authorize writes and fabricate fills.** Order-book and balance failures are absorbed (`bot/index.js:51-90`); failed reconciliation returns only from its helper (`:94-103`) and the tick continues. Unpaginated open-order absence marks full submitted quantity filled (`:105-125`).
4. **P0 — placement is not intent-first or ambiguity-safe.** Millisecond request IDs are neither durable nor proven idempotent (`bot/dextrade.js:128-139`). Missing IDs become `"undefined"`; a timeout logs and permits later batch/next-tick exposure (`bot/index.js:208-230`).
5. **P0 — restart and lifecycle races can duplicate or orphan exposure.** State is process-local; startup does not adopt remote orders; `tickInProgress` does not serialize HTTP stop/config/rebalance with placement.

## High money-contract and evidence gaps

- Missing orders are booked at full submitted price/quantity; order history is unused, so partial fills, cancellations, execution prices and fees are not reconciled.
- The no-database rule is incompatible with crash-safe at-most-once placement unless dex-trade supplies a proven unique idempotent/recoverable client reference. No such contract is established.
- Four-decimal price/volume and the 5 USDT minimum are hard-coded without consumed symbol metadata; admission checks can differ from transmitted rounded amounts.
- Strategy validation is finite-number-only and lacks schema ownership, integer count rules and order/batch/account exposure caps.
- PnL uses floating-point submitted values and ignores exact fills and fee currency.
- No repository test script or CI workflow exists; live pagination, cancellation finality, fees and timeout-after-acceptance semantics remain unproved.

## Executed evidence

| Gate | 2026-09-10 result |
|---|---|
| Branch/remote identity and clean start | **PASS** — local and remote `10db348c947afcb508ea533ddf246603ed56beff`; clean worktree |
| Prior-review delta/runtime tree identity | **PASS** — documentation-only delta; `bot/` and `src/` trees unchanged |
| `npm run build` | **PASS** — Webpack 5.99.9, 45.7 KiB production bundle; stale Browserslist-data warning |
| `node --check bot/*.js bot/strategies/*.js` | **PASS** — all 9 files |
| `npm audit --omit=dev` | **FAIL** — 2 root vulnerable packages (1 high, 1 low) |
| `npm audit --omit=dev --prefix bot` | **FAIL** — 6 bot vulnerable packages (3 high, 2 moderate, 1 low) |
| Tracked Markdown link check | **FAIL (inherited docs snapshot)** — 20 unresolved links, all under `Nexus API docs/`; this subtree did not change |
| `git diff --check` before edits | **PASS** |
| Configured tests / checked-in CI | **ABSENT** |
| Live exchange behavior | **NOT RUN** by safety scope |

Audit counts are unchanged from 2026-09-09 and are not new failures. The broad link failures are inherited references into a partial Nexus API documentation snapshot, not a regression in this review's authoritative root documents.

## Repair handoff

Implement and independently review in this order:

1. Default-deny write enablement and an isolated network-denying `node:test`/CI seam.
2. Authenticated exact-origin control plus server-owned schemas and explicit exposure caps.
3. Typed fresh/complete read evidence that places the bot in a visible hold on transport, schema or pagination uncertainty.
4. Durable or exchange-idempotent attributable placement/cancellation with fault injection at every await/crash boundary.
5. One serialized lifecycle queue and startup reconciliation before any placement.
6. Exact symbol metadata, decimal-safe fills/fees/PnL and capped sandbox/test-account acceptance.
7. Dependency remediation only under the complete behavior/build/wallet gate.

Do not use credentials or meaningful capital until every P0 exit in `DEVELOPMENT_PLAN.md` passes and target-exchange semantics are evidenced.
