# Development and Architecture Review — 2026-09-25

## Decision

**Reviewed source:** `e2e849cafd767ccd306b4144da2dcd5f67be8460` on `main`.

**Verdict: unsafe for unattended trading or meaningful capital.** No implementation repair landed after the 2026-09-23 reviewed source. The target commit changes only the prior architecture, development-plan and review documents; `bot/`, `src/`, manifests and lockfiles are unchanged. Every P0 release gate remains open, and fresh execution found an additional exact-unit mismatch plus an idle-prefetch lifecycle gap.

This review made no dex-trade request, loaded no real credential, changed no runtime or dependency, and created/cancelled no exchange order. Safety probes used synthetic credentials, intercepted exchange/process boundaries, an ephemeral `127.0.0.1` listener and scratch output under the active Hermes profile. No commit, push, staging, reset, clean or stash operation was performed.

## Baseline and development delta

At review start:

- `git rev-parse HEAD` returned `e2e849cafd767ccd306b4144da2dcd5f67be8460`.
- `git status --short --branch` returned `## main...origin/main` plus only `?? vision.md`.
- The real index tree was `9d6c504099e3d919eb5a07aa9b268c6753d76d1e`.
- The locally recorded `origin/main` matched HEAD (`0` ahead, `0` behind); no network fetch was used for the offline source assessment.
- `git diff --name-status 64185101218bf3b8b47666af07d78720b25cc9f7..e2e849cafd767ccd306b4144da2dcd5f67be8460` returned only:
  - `M ARCHITECTURE.md`
  - `M DEVELOPMENT_PLAN.md`
  - `A DEVELOPMENT_REVIEW_2026-09-23.md`
- Runtime trees are unchanged: `bot/` = `cfe5e308bbc01fc5b55329bc4378ac449720a70d`; `src/` = `eac7280ff4119b667d85fa2be66d3062fc35de58`.
- The untracked `vision.md` was preserved and reviewed at SHA-256 `733a95bbb8d59d0e55acebe486a3bbbca82b70f89d66bbe0a3b50c333685c5d0`. It is context, not part of the target commit or this review's intended publication scope.

The vision correctly narrows the purpose to operator-controlled, bounded liquidity infrastructure rather than an on-chain AMM, pooled-custody product or investment guarantee. Its operator custody, no-underwriting, loss-limit, exact-money, reconciliation-first and transparent-evidence constraints are now reflected in the authoritative architecture and plan. That alignment is documentation only; current code does not enforce it.

## Fresh commands and results

| Command / probe | Result |
|---|---|
| `npm run build -- --output-path /home/brutus/.hermes/profiles/principal-dev/cache/scratch/amm-build-2026-09-25-e2e849c` | **PASS** — Webpack `5.99.9`; 45.7 KiB minimized `app.js`; compiled in 736 ms. Warned that Browserslist data is 16 months old. |
| `node --check` over tracked `bot/*.js` and `bot/strategies/*.js` | **PASS** — `syntax_checked=9`. |
| root `npm test` | **FAIL / absent** — exit 1, `Missing script: "test"`. |
| `cd bot && npm test` | **FAIL / absent** — exit 1, `Missing script: "test"`. |
| `git ls-files '.github/**' '*test*' '*spec*'` | Empty: no tracked CI, test or spec candidate. |
| root `npm ls --depth=0 --omit=dev` | **PASS** — `nexus-module@1.1.11`, `react-redux@8.1.2`, `redux@4.1.2`. |
| bot `npm ls --depth=0 --omit=dev` | **PASS** — `axios@1.13.5`, `cors@2.8.6`, `dotenv@16.6.1`, `express@4.22.1`. |
| root and bot `npm audit --offline --omit=dev` | Exit 0, `found 0 vulnerabilities` in each package. Cache-only evidence does not close the 2026-09-21 online advisory failures. |
| `gh run list --repo distordialabs-brutus/amm_nxs-usdt_dextrade --commit e2e849cafd767ccd306b4144da2dcd5f67be8460 --limit 20 --json ...` | `[]`: no GitHub Actions run exists for the exact reviewed SHA. |
| Scratch probe, scenarios `server import balance negative-open missing-id cancel-cardinality stop-race restart exact-units` | **Executed all nine scenarios** against unchanged production callers with only external/process boundaries replaced. Probe SHA-256: `3f424202f25534f12432dc23a2ec225d6182c35c84dba5b16e14d558854fdf6b`. |
| Live exchange semantics | **NOT RUN** by scope and release order. |

Scratch probe path: `/home/brutus/.hermes/profiles/principal-dev/cache/scratch/amm-safety-probe-2026-09-25.js`. It is review evidence, not a tracked or collected test.

### Fresh safety outputs

- **Attacker-origin control:** `/api/start`, `/api/config`, `/api/rebalance` and `/api/stop` each returned 200 with `Access-Control-Allow-Origin: *` and invoked the fake controller. `numGrids: 1000000` and unknown `unknownExposureKey: 7` reached start/config unchanged.
- **Import effects:** requiring `bot/index.js` attempted one listener, one interval, two signal handlers and initial ticker/book/balance reads (`1/1/1`).
- **Failed balances:** all three observed balance reads failed (an overlapping idle prefetch, tick refresh and rebalance refresh), yet the real controller path placed two orders from cached balances.
- **Negative open-list evidence:** an empty open-order list changed seeded A/B orders to `filled` and booked buy volume `10`, sell volume `10`, buy cost `10`, sell revenue `12`, realized PnL `2`, fees `0` without a fill identity.
- **Missing IDs:** two `{}` create responses made two remote create calls but left one local key, literal `undefined`; the second order overwrote the first.
- **Cancellation cardinality:** one successful result for requested A/B/C caused all three local rows to become `cancelled`.
- **Stop barrier:** when placement was pending, stop returned state `stopped`, no known order, zero cancellation and no trading interval. After acceptance resolved, `accepted-after-stop` was locally `open`, cancellation remained zero and `start()` installed a new interval.
- **Restart:** one process held one managed order; a fresh process loaded zero. There is no durable intent/outcome state.
- **Exact units:** raw `price=2`, `volume=2.50006` produced controller notional `5.00012`, which passes `5.00015` available. The real adapter formatted wire volume `2.5001`, making wire notional `5.0002`, greater than the checked balance.

## Findings in financial-risk order

### 1. P0 — terminal stop and idle work are not one lifecycle

`start()` sets `running`, awaits the first `tick()`, then unconditionally installs an interval (`bot/index.js:275-292`). `stop()` independently clears the current interval, snapshots only already-recorded IDs and reports `stopped` (`bot/index.js:295-317`). A placement is not recorded until its await resolves (`bot/index.js:208-230`). The fresh barrier reproduced an accepted open order and a newly installed interval after terminal stop.

Idle prefetch is also outside lifecycle serialization. `prefetchSnapshotWhenStopped()` checks `running` once, then awaits market and balance reads (`bot/index.js:34-47`). Start can win after that check, allowing the idle call to continue mutating the same market/balance projection concurrently with the first tick. The fresh failed-balance run observed that overlap.

**Required exit:** one generation-aware admission/transition protocol covers idle prefetch, tick, start, config, rebalance, stop and shutdown. Stop closes admission first, invalidates the generation, joins the active transition, and returns plain `stopped` only when no admitted work can issue a later private call/install a timer and all attributable orders are positively terminal. Unknown acceptance produces a disabled, quantified hold.

### 2. P0 — mutation is unauthenticated, default-on with credentials, and unbounded

`bot/server.js:31-36,56-105` installs wildcard CORS and exposes every mutation without a capability. Validation accepts arbitrary finite numeric keys and does not enforce strategy metadata, integer counts or economic caps (`bot/server.js:16-29`). `src/App/Main.js:64-68,136-164` sends no runtime capability. There is no independent `TRADING_ENABLED` switch, origin policy, per-order/batch/inventory/unresolved-exposure cap, or vision-required loss cap.

Loopback binding is containment, not caller authority. Browser private-network behavior is neither tested nor a server-side safety control.

**Required exit:** mutation defaults disabled and rejects missing policy before controller/private-client construction. Require exact configured browser origin plus a non-bundled high-entropy capability; define an authenticated no-Origin operator path. Closed schemas and independently enforced order, batch, inventory, unresolved-exposure and loss caps must reject attacker origin, missing/wrong/duplicate credential, unknown field, fractional count, million-grid, oversized body and over-cap cases with zero controller, journal, state or exchange calls.

### 3. P0 — stale or failed reads authorize writes

`fetchBalances()` catches errors and returns no typed result (`bot/index.js:75-90`). Tick and rebalance both call it, but rebalance then consumes cached `state.balances` regardless (`bot/index.js:169-172,246-258`). The fresh probe failed every balance read and still placed two orders. `fetchMarket()` likewise falls back to last trade after order-book failure (`bot/index.js:51-71`). `getOpenOrders()` supplies no pair/cursor/completeness evidence (`bot/dextrade.js:156-165`).

**Required exit:** each financial transition consumes one pair-bound, schema-valid, fresh, complete evidence bundle containing symbol metadata, non-crossed book, balances, orders/history and pagination/range proof. Any missing member holds with zero cancellation, placement or accounting mutation. Cached values remain display-only.

### 4. P0/P1 — absence fabricates fills and profit

`reconcileOrders()` converts absence from the open-order set into a full fill and books submitted volume/price (`bot/index.js:94-125`). The existing history client is unused. Fresh execution invented 2 USDT realized PnL without any fill ID, execution quantity/price, fee or complete history.

**Required exit:** open-list absence remains unresolved. Only immutable positive fill/closed evidence keyed by exchange order and fill identity can change reservations or accounting. Tests must cover partial-fill-then-cancel, multiple fills, duplicate evidence, NXS/USDT fees, unequal decimals, malformed/truncated history and zero checked entities. Incomplete reconciliation is unhealthy, never balanced.

### 5. P0 — writes lack durable attributable intent and restart recovery

Private `request_id` is `String(Date.now())` immediately before each call (`bot/dextrade.js:114-176`), with no durable uniqueness or proven exchange lookup/idempotency contract. Missing IDs become literal `undefined`; errors are logged and the batch continues (`bot/index.js:208-230`). All order/PnL state is process-local (`bot/state.js`), and fresh-process execution loaded zero after another process seeded one order.

A timeout or crash after acceptance is therefore indistinguishable from rejection. Blind retry can duplicate exposure; continuing can strand it. Startup performs no complete adoption/reconciliation before placement.

**Required exit:** persist frozen pair, side, exact wire units/notional, strategy/session generation, unique reference and reservation before transport. Timeout, malformed success, interruption and identity-persistence failure become durable `outcome_unknown`, stop the batch and block replacement. Separate-process fault tests must inject every intent/submit/accept/parse/identity/finalize/restart boundary and assert exact remote-attempt count, identity, reservation, journal state and admission result.

### 6. P0 — cancellation evidence is discarded

`cancelAll()` omits failed IDs (`bot/dextrade.js:181-193`); rebalance and stop ignore returned identity/cardinality and mark every requested row cancelled (`bot/index.js:161-166,301-310`). Fresh A/B/C execution returned one success and finalized all three.

**Required exit:** one ID-correlated typed result per request: `confirmed_cancelled`, `still_open` or `outcome_unknown`. Only positive authoritative evidence releases exposure; timeout-after-acceptance stays held through stop/restart until resolved.

### 7. P0 — policy, reservation and wire amounts disagree

The controller rounds price to four decimals but checks minimum/balance and stores/reserves raw strategy volume (`bot/index.js:175-221`). The adapter independently applies `volume.toFixed(4)` (`bot/dextrade.js:128-139`). Fresh execution proved a raw amount can pass balance admission while the transmitted rounded amount exceeds it. Floating-point PnL, hard-coded precision and a hard-coded 5 USDT minimum compound the mismatch.

**Required exit:** fetch and validate symbol precision/minimum metadata; quantize strategy proposals once into integer-scaled or exact decimal units; derive immutable wire strings from those units. Policy, minimum, balance, risk caps, reservation, journal, API payload and reconciliation must consume the same amount. Add below/exact/above rounding/minimum/balance cases with unequal asset decimals.

### 8. P0/P1 — the operator-risk contract in `vision.md` is not executable

The source preserves operator exchange-account custody, but no code enforces explicit write enablement, independent capital limits, maximum loss/drawdown, unresolved-outcome capacity or human disposition. Dashboard status is an ephemeral projection, not inspectable durable evidence. README still calls in-memory state intentional and claims reconciliation each tick, which contradicts executed restart and absence behavior (`README.md:215-223`). It also points to an obsolete “latest” review (`README.md:15-18`).

**Required exit:** implement and test fail-closed policy, durable held reasons/evidence, operator disposition audit and truthful status. Correct README only with the same accepted implementation batch so maintained setup claims do not overtake code.

### 9. P1 — no engineering gate, CI evidence or live semantic evidence exists

Both packages lack a test script; no tracked test or `.github` workflow exists; GitHub has no run for the exact SHA. Import itself performs listener/timer/signal/private-read work, obstructing isolation. Build and syntax prove only packaging/parsing. Offline audits are cache-dependent. No approved target-account evidence establishes pair filtering, pagination, precision/minimums, fee schema, cancellation finality, client-reference lookup/idempotency or timeout-after-acceptance behavior.

**Required exit:** one root command collects production-path tests under mandatory network denial and CI runs it on every push. After Batches A-E pass, separately approve a capped disposable target-account matrix. Do not weaken ambiguity holds if dex-trade lacks direct attributable recovery.

## Positive controls actually verified

- HTTP listener remains bound to `127.0.0.1` in `bot/index.js:338-339`.
- Exchange HTTP remains centralized in `bot/dextrade.js`.
- Strategy modules remain I/O-free target generators.
- Scheduled ticks retain a non-overlap guard, though it is not lifecycle serialization.
- Frontend production build and all nine bot syntax checks pass.
- Installed direct production dependencies resolve.
- The source and real index were unchanged before documentation edits; untracked `vision.md` was preserved.

These controls reduce exposure but do not establish production readiness.

## Prioritized coder handoff

1. **Batch A only — containment/gate:** split `bot/index.js` into composition plus injected `bot/controller.js`/`bot/config.js`; make server construction import-safe; add root-collected tests, mandatory network denial and `.github/workflows/ci.yml`; default `TRADING_ENABLED=false`. Include idle prefetch in the admission design. Exit requires zero import side effects and zero private calls under incomplete policy.
2. **Batch B — authority/lifecycle:** exact origin + non-bundled capability, closed schemas, order/batch/inventory/unresolved/loss caps, and one generation-aware transition protocol for every caller. Reproduce the stop barrier; plain stop cannot leave an accepted/unknown order or later timer.
3. **Batch C — typed reads:** symbol metadata, pair/freshness/schema/completeness and pagination proof; remove last-price and cached-balance authorization. Reproduce every failed-read and empty-open-list case through the real controller with zero financial effects.
4. **Batch D — exact durable writes:** quantize once before policy, freeze wire units, persist intent/reservation before transport, hold every ambiguous outcome, enforce cancellation cardinality, and recover in separate processes without a duplicate remote write. The `2 × 2.50006` boundary is a mandatory regression.
5. **Batch E — startup/fills/accounting:** start held; adopt/reconcile every attributable and unknown order; consume fill IDs, exact base/quote units and fees; prove partial/duplicate/unequal-decimal accounting and unhealthy incomplete reconciliation.
6. **Batch F only after A-E acceptance:** capped disposable dex-trade semantics, online audit remediation under the full gate, Nexus Wallet production-install/control smoke, runbook and incident exercises.

Do not start with dependency upgrades, strategy expansion, backtesting or live trading. Those do not contain the demonstrated fund-loss paths.

## Reviewed source hashes

Git identities:

| Object | Hash |
|---|---|
| Source commit | `e2e849cafd767ccd306b4144da2dcd5f67be8460` |
| Real index tree before review edits | `9d6c504099e3d919eb5a07aa9b268c6753d76d1e` |
| `bot/` tree | `cfe5e308bbc01fc5b55329bc4378ac449720a70d` |
| `src/` tree | `eac7280ff4119b667d85fa2be66d3062fc35de58` |
| root `package.json` blob | `ccd923dd59e43f885e461269b55de98abba1bc00` |
| root lock blob | `5b1f7a84094cdddbefe27356d2929598d2f7b41b` |
| bot `package.json` blob | `ab5002498a0b9819c2c56a6a975c1976f96a9bc6` |
| bot lock blob | `4a678bb07ca68b45d6d941bc5325aeb5af7309d8` |

SHA-256:

| File | SHA-256 |
|---|---|
| `bot/dextrade.js` | `814f5290b171473b5c0974e76632eb2b3d635735af5fed8245108c5938b58a3e` |
| `bot/index.js` | `9540042742edbd722f0134ac7b719e3aecd916f0feeaf2a45bdbb9768ed0d970` |
| `bot/logger.js` | `4f481132588c8050a0859f870d3f465e245379ee8f89ceeeca6e500b7d3fd623` |
| `bot/server.js` | `0daea7cea2ffe5d607c6e818aaa7f849943dcabfb955bb586598dc87f430f075` |
| `bot/state.js` | `b32a933dc7af9ba0f832613e93eee6aac967de3a4337f6a8d0d88f6932b17499` |
| `bot/strategies/constantProduct.js` | `243e9dd67a89adb832c45ce2871f4d205dc1cb785fafb455193be5ce33f51021` |
| `bot/strategies/grid.js` | `568cc6b095aa92568d05a4b3e0c1bf0b4fd58b931fbdb17ba3804b7f3578e48d` |
| `bot/strategies/index.js` | `4efa7326cb8bf587efee69481bde3b11a7577720150386dd84e24cbca0532f64` |
| `bot/strategies/spreadMaker.js` | `e05084b3cb2e86ca718cc875149b1f43bf1b420b73a3aaa2e451ef6bf95dcaa7` |
| `src/App/Main.js` | `3ca43136336354c897babe54b3a7e2c9548baf548c40223b481bbd4617b8b830` |
| root `package.json` | `ea594e4edc1baebc6f06598271423c469f13aec05100be6ae3daa6f88f2d79df` |
| root lock | `cbe5fa7817ff8b09a60fa814ad572ab567c19331b38950fedfb18d53ab00b4f4` |
| bot `package.json` | `7c01f8bf9a332eea04fe30a53df79ab7a9e70e3cc1874d58c3d791999b89dc2b` |
| bot lock | `837b4c5677cfd6fe90e1d15396f920d4d3d4818bbeb235b5fab448391e06453c` |
| `nxs_package.json` | `dbf72e295506511f268c99542117a3fa677875b8796a04b70a998e54ed7cbcbe` |
| untracked `vision.md` | `733a95bbb8d59d0e55acebe486a3bbbca82b70f89d66bbe0a3b50c333685c5d0` |

The dated source hashes identify the exact implementation assessed even though the authoritative documentation is now modified in the working tree. This review was not committed or pushed.
