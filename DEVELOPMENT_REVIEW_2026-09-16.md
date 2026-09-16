# Development and Architecture Review — 2026-09-16

## Baseline, delta and verdict

**Source HEAD:** `01038ec26f8e19bb11e4eb11a7b8c1e1a2708257` on `main`, equal to freshly fetched `origin/main` (`0` ahead, `0` behind).

The only delta from the 2026-09-15 review baseline (`21c575206091cfb0d7117f1629a58f625d9909a9`) is the prior documentation publication: `ARCHITECTURE.md`, `DEVELOPMENT_PLAN.md` and `DEVELOPMENT_REVIEW_2026-09-15.md`. There is **no executable or dependency-manifest delta**. The current `bot/` tree is `cfe5e308bbc01fc5b55329bc4378ac449720a70d` and `src/` is `eac7280ff4119b667d85fa2be66d3062fc35de58`, identical to the prior baseline.

**Verdict: not ready for unattended trading or meaningful capital.** This review did not discover a new runtime regression because runtime did not change. It converted the previously blocked production-path probes into fresh executable evidence and confirmed that the documented P0 failures are reachable through the real server/controller modules. No real credential, exchange request, order, cancellation or trade was used.

## Critical paths — freshly executed confirmation of existing findings

All probes used the production modules with only process, HTTP or exchange boundaries replaced by in-memory fakes. They were run from `/tmp`; none is a checked-in test or release gate.

1. **Every mutation route crosses the unauthenticated wildcard-CORS boundary.** An ephemeral loopback server created from `bot/server.js` received requests with `Origin: https://attacker.invalid`, no authorization header and a fake controller. `/api/start`, `/api/config`, `/api/rebalance` and `/api/stop` each returned HTTP 200, each returned `Access-Control-Allow-Origin: *`, and each invoked its controller method. The config request containing `numGrids: 1000000` and unknown key `unknownExposureKey` was accepted and forwarded unchanged. This corroborates `bot/server.js:34-36,56-105`; it does not depend on browser private-network policy.
2. **Read uncertainty reaches writes and fabricated accounting.** In an isolated import of the real `bot/index.js`, four fake exchange orders were placed. A subsequent empty open-order list changed all four to `filled` and booked `20 NXS` bought, `20 NXS` sold and `45 USDT` realized PnL from submitted values alone. In a separate transition in the same harness, two injected balance-read failures were logged/absorbed and the controller still made four placement calls using the retained `100000 NXS / 100000 USDT` snapshot. These are direct executions of `bot/index.js:75-90,94-125,246-256`.
3. **Missing placement identity collapses multiple possible remote actions into one local row.** Four `createLimitOrder` calls returning `{}` produced one open `state.managedOrders.undefined` entry, retaining only the last order. The remaining placement batch was not aborted. This executes `bot/index.js:208-230` and demonstrates why response validation plus an outcome-unknown exposure hold must precede any later batch item.
4. **Partial cancellation is converted to universal local success.** The real `DexTradeClient.cancelAll()` attempted IDs `A`, `B`, `C`, suppressed the injected failure for `B`, and returned only two successes. Separately, the real controller caller received one success for three requested IDs (`undefined`, `cancel-A`, `cancel-B`) and marked all three `cancelled`. This executes both sides of `bot/dextrade.js:184-193` and `bot/index.js:301-310`; the same caller pattern remains at rebalance (`bot/index.js:161-166`).
5. **The composition seam is still absent.** With HTTP and exchange modules intercepted before loading, merely importing `bot/index.js` attempted a listener on `127.0.0.1:17442`, installed one interval, added one SIGINT and one SIGTERM handler, and invoked ticker, order-book and balance reads. This is executable evidence that controller tests cannot safely import the production composition module without loader-level interception.

These are fresh demonstrations of defects already documented on 2026-09-15, not newly introduced findings.

## Other open architecture gaps (source-confirmed, not rebranded as new)

- State, order ownership, PnL and request references remain process-local; startup has no adoption/recovery scan.
- Millisecond request IDs are not durable or proven idempotent, and concurrent lifecycle/private calls are not serialized.
- Open-order enumeration has no pair, cursor or completeness result; `getOrderHistory()` remains unused.
- Hard-coded four-decimal formatting and 5 USDT admission use floating point and do not consume authoritative symbol/fee metadata.
- The frontend sends no control credential because the server defines no authenticated control contract.

## Fresh non-trading gates

| Gate | Result |
|---|---|
| Fetch and branch readback | **PASS** — HEAD = `origin/main` = `01038ec26f8e19bb11e4eb11a7b8c1e1a2708257`; `0 0` |
| Runtime delta check | **PASS** — `bot/` and `src/` tree IDs match the 2026-09-15 review baseline; only three documentation files changed |
| `npm run build` | **PASS** — Webpack `5.99.9`, 45.7 KiB production bundle, compiled in 750 ms; stale Browserslist-data warning |
| `node --check` over tracked bot/strategy JavaScript | **PASS** — 9 files |
| Root `npm ls --omit=dev --depth=0` | **PASS** — `nexus-module@1.1.11`, `react-redux@8.1.2`, `redux@4.1.2` |
| Bot `npm ls --omit=dev --depth=0` | **PASS** — `axios@1.13.5`, `cors@2.8.6`, `dotenv@16.6.1`, `express@4.22.1` |
| Root `npm audit --omit=dev` | **FAIL** — 2 vulnerable entries: 1 high, 1 low |
| Bot `npm audit --omit=dev` | **FAIL** — 6 vulnerable entries: 3 high, 2 moderate, 1 low |
| Root and bot `npm test` | **FAIL** — both report `Missing script: "test"` |
| Isolated HTTP boundary probe | **FAIL as a safety criterion** — all four unauthenticated attacker-origin mutations returned 200 and invoked the controller |
| Isolated controller/adapter fault probe | **FAIL as a safety criterion** — fabricated fills/PnL, stale-balance placements, missing-ID collapse and false cancellation finalization reproduced |
| `git diff --check` before documentation edits | **PASS** |
| Live exchange semantics | **NOT RUN** by safety scope |

## Repair order and executable exits

1. **Build the import-safe seam and default-deny control boundary first.** Extract controller construction from process startup. Add independent `TRADING_ENABLED=false`, exact origin, control capability and hard exposure caps. Check these in one middleware before any controller call. **Exit:** imports create zero listener/timer/signal/network effects; replaying the attacker-origin probe returns 4xx/503 for every mutation with zero controller calls; the million-grid/unknown-key request is rejected.
2. **Install one collected, network-denying root gate.** Convert the temporary probes into deterministic `node:test` fixtures with fake clock/ID/exchange boundaries and run them in CI alongside clean installs, build, syntax, audit policy and whitespace checks. **Exit:** root `npm test` exists, fails on an unexpected network attempt, and CI runs it from a clean checkout.
3. **Make uncertainty a visible hold before more write capability.** A failed/malformed/incomplete book, balance or order enumeration must end the transition with zero cancel/place/PnL effects. Missing orders require attributable fill/cancel evidence. **Exit:** the empty-open-list fixture leaves all four orders unresolved with zero PnL; both failed balance reads produce zero placements and an operator-visible reason.
4. **Make writes attributable and cancellation cardinality exact.** Validate canonical order IDs, abort on the first ambiguous placement, reserve its possible exposure, and return one typed cancellation result per requested ID. **Exit:** four missing-ID responses cannot produce later placement calls; one-of-three cancellation success leaves two orders open/outcome-unknown rather than cancelled.
5. **Resolve restart durability and serialization.** Approve a minimal intent journal or prove unique exchange idempotency/direct lookup, then serialize tick/start/stop/config/rebalance. **Exit:** crash and await-boundary tests demonstrate one attributable remote action, retained unknown exposure, safe stop and startup adoption before placement.

Dependency remediation remains compatibility-gated behind these behavior tests. Sandbox/test-account validation is still required for pagination, partial fills, fees, cancellation and timeout-after-acceptance semantics before any production-readiness claim.
