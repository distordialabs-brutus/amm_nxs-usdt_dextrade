# Development and Architecture Review — 2026-09-28

## Decision

**Reviewed source:** `0040f42736f61dead2a612b4ed9295816ecd093a` on `main`.

**Verdict: unsafe for unattended trading or meaningful capital.** HEAD is unchanged from the supplied source SHA and contains only the prior review's documentation delta over `e2e849cafd767ccd306b4144da2dcd5f67be8460`. Runtime source, manifests and lockfiles are unchanged; every P0 remains open. Fresh execution reconfirmed the nine prior offline failure scenarios and added a tenth: `stop(false)` reports terminal `stopped` while two exchange orders remain locally open and no cancellation is attempted.

This review made no dex-trade request, loaded no real credential, changed no runtime or dependency, and created/cancelled no live exchange order. Probes used synthetic credentials, intercepted exchange/process boundaries, an ephemeral `127.0.0.1` listener and scratch files under the active Hermes profile. No staging, reset, clean, stash, commit or push was performed.

## Baseline and development delta

At review start:

- `git rev-parse HEAD` returned `0040f42736f61dead2a612b4ed9295816ecd093a`.
- `git status --short --branch` returned `## main...origin/main` plus only `?? vision.md`.
- The real index tree was `087093fafaede7fee40e8f5eaf6c263e35759cdb`, equal to `HEAD^{tree}`.
- Locally recorded `origin/main` matched HEAD. The sanitized origin was `https://github.com/distordialabs-brutus/amm_nxs-usdt_dextrade.git`.
- `git diff --stat 0040f42736f61dead2a612b4ed9295816ecd093a..HEAD` was empty.
- The target commit's parent is `e2e849cafd767ccd306b4144da2dcd5f67be8460`; its delta modifies only `ARCHITECTURE.md`, `DEVELOPMENT_PLAN.md` and adds `DEVELOPMENT_REVIEW_2026-09-25.md`.
- Runtime trees remain `bot/` = `cfe5e308bbc01fc5b55329bc4378ac449720a70d` and `src/` = `eac7280ff4119b667d85fa2be66d3062fc35de58`.
- The untracked `vision.md` was preserved at SHA-256 `733a95bbb8d59d0e55acebe486a3bbbca82b70f89d66bbe0a3b50c333685c5d0`. It remains review context, not source HEAD or intended staging scope.

## Commands and evidence

| Command / probe | Result |
|---|---|
| `npm run build -- --output-path .../amm-build-2026-09-28-0040f42` prefixed by scratch cleanup | **BLOCKED before build execution** because the combined command contained recursive deletion and unattended approval was unavailable. Per review constraints, the denied probe was not rerouted. The unchanged runtime's 2026-09-25 build pass is historical evidence only, not a fresh build result. |
| `node --check` over `git ls-files 'bot/*.js' 'bot/strategies/*.js'` | **PASS** — `syntax_checked=9`. |
| root `npm test` | **FAIL / absent** — exit 1, `Missing script: "test"`. |
| `cd bot && npm test` | **FAIL / absent** — exit 1, `Missing script: "test"`. |
| `git ls-files '.github/**' '*test*' '*spec*'` | Empty — no tracked CI/test/spec candidate. |
| root `npm ls --depth=0 --omit=dev` | **PASS** — `nexus-module@1.1.11`, `react-redux@8.1.2`, `redux@4.1.2`. |
| bot `npm ls --depth=0 --omit=dev` | **PASS** — `axios@1.13.5`, `cors@2.8.6`, `dotenv@16.6.1`, `express@4.22.1`. |
| root and bot `npm audit --offline --omit=dev` | Exit 0, `found 0 vulnerabilities` in each. Cache-only evidence does not close prior online findings. |
| `gh run list --repo distordialabs-brutus/amm_nxs-usdt_dextrade --commit 0040... --limit 20 --json ...` | `[]` — no GitHub Actions run exists for the exact source SHA. |
| `node -e` using `node:sqlite` | In-memory create/insert/select succeeded on Node `v22.23.2`, but Node emitted `ExperimentalWarning: SQLite is an experimental feature`. This is host capability evidence, not approval of a production journal adapter. |
| Scratch probe scenarios `server import balance negative-open missing-id cancel-cardinality stop-race restart stop-no-cancel exact-units` | **PASS as a defect-reproduction probe** — all ten expected unsafe outcomes were asserted against unchanged production callers with only external/process boundaries replaced. Probe SHA-256: `a466e9dc57ea7db287106a3114f3795140048d175e46229f9665b147f74fe7db`. |
| Live exchange semantics | **NOT RUN** by scope and release order. |

Scratch probe path: `/home/brutus/.hermes/profiles/principal-dev/cache/scratch/amm-safety-probe-2026-09-28.js`. It is review evidence, not a tracked or collected suite.

### Fresh probe outputs

- **Attacker-origin control:** `/api/start`, `/api/config`, `/api/rebalance` and `/api/stop` each returned 200, exposed `Access-Control-Allow-Origin: *`, and invoked the fake controller. `numGrids: 1000000` and `unknownExposureKey: 7` reached start/config unchanged.
- **Import effects:** requiring `bot/index.js` attempted one listener, one interval, two signal registrations and initial ticker/book/balance reads (`1/1/1`).
- **Failed balances:** three balance reads failed (idle prefetch, tick and rebalance), yet two orders were placed from cached balances.
- **Negative open-list evidence:** an empty open list changed seeded A/B to `filled` and booked buy volume `10`, sell volume `10`, buy cost `10`, sell revenue `12`, realized PnL `2`, fees `0` without fill identity.
- **Missing IDs:** two `{}` responses caused two create calls but left one local key, literal `undefined`; the second order overwrote the first.
- **Cancellation cardinality:** one returned success for requested A/B/C caused all three local rows to become `cancelled`.
- **Stop race:** stop returned `stopped` with no known order, zero cancellations and no trading interval beyond idle prefetch. After acceptance resolved, `accepted-after-stop` was `open`, cancellation remained zero and `start()` installed a second interval.
- **Restart:** one process seeded one managed order; a fresh process loaded zero.
- **Stop without cancellation:** after two accepted orders, `stop(false)` returned `stopped`, attempted zero cancellations and retained `open-buy` and `open-sell` as `open`.
- **Exact units:** controller price `2` and raw volume `2.50006` produced raw notional `5.00012`, passing available USDT `5.00015`; the real adapter serialized `2.0000 × 2.5001 = 5.0002`, greater than the checked balance.

## Findings in financial-risk order

### 1. P0 — terminal stop is false under two independent paths

The existing await race remains: `start()` can accept and record an order and install a timer after `stop()` has returned terminal. The newly executed `stop(false)` path is a separate direct defect: the public route forwards a caller-controlled boolean, and `bot/index.js:295-317` unconditionally sets `state.status = 'stopped'` even when known open orders are deliberately left live.

**Required exit:** replace the boolean with a closed stop mode. `cancel-and-stop` may return `stopped` only after no admitted work can issue a later private call/install a timer and every attributable order is positively terminal. An explicit `disable-and-hold` may leave orders open only while reporting a quantified `held` state and retaining all reservations. Unknown modes and ambiguous booleans fail closed.

### 2. P0 — unauthenticated, default-on, unbounded mutation remains executable

`bot/server.js:31-36,56-105` still uses wildcard CORS and no capability. Finite-number validation accepts unknown keys and ignores strategy min/max/integer contracts and economic caps. The frontend supplies no runtime capability.

**Required exit:** default-disabled mutation; exact configured origin; non-bundled high-entropy capability; explicit authenticated no-Origin operator policy; closed request schemas; independent order, batch, inventory, unresolved-exposure and loss caps. Every refusal asserts zero controller, journal, state and exchange effects.

### 3. P0 — failed/stale/incomplete reads still authorize writes and fabricate state

Balance failures are absorbed and cached balances authorize placement. Order-book failure can fall back to last trade. Open-order reads carry no pair/page/completeness proof. Open-list absence fabricates fills/PnL from submitted terms.

**Required exit:** one pair-bound, schema-valid, fresh and complete evidence bundle per financial transition; any missing member holds. Open-list absence remains unresolved until positive closed/fill evidence proves terminal state. Cached data is display-only.

### 4. P0 — placement and cancellation lack attributable durable outcomes

Millisecond references are not durably unique or proven exchange-idempotent. Missing IDs collapse into `undefined`; failed placements do not stop a batch; cancellation results lose cardinality and all requested IDs are finalized locally. Process restart loses every intent, order and reservation.

**Required exit:** persist exact intent/reservation before transport; treat timeout, malformed success, interruption and identity-persistence failure as durable `outcome_unknown`; abort later batch work; return one typed cancellation outcome per ID; recover every non-terminal row before placement.

### 5. P0 — repository instructions contradict required financial durability

`CLAUDE.md:29` says all bot state is in-memory and prohibits a database. That is incompatible with restart-safe intent, identity, reservation and unknown-outcome retention. A documentation correction to the protected file was attempted and denied by approval policy; it was not retried or bypassed.

The architecture now selects the durable-intent path: ordinary projections may stay in memory, but financial safety state requires a dedicated transactional journal. The storage engine is not yet approved. Experimental `node:sqlite` availability on one review host is insufficient.

**Required exit:** a maintainer approves correction of the protected instruction and approves an ADR naming one audited adapter, durability settings, filesystem/runtime support, corruption behavior, backup/restore and packaging. Without both approvals, Batch D and trading remain blocked; no in-memory or ad-hoc JSON fallback is allowed.

### 6. P0 — one process queue would not prevent two-process writes

The plan previously required one controller queue but did not explicitly fence multiple bot processes sharing future durable state. A second launch, stale lease holder or restored copy could otherwise replay or mutate under independent in-process locks.

**Required exit:** obtain one OS-enforced exclusive writer lock or transactional lease with monotonic fencing before recovery/private calls. Two real processes against one journal must prove one writer, exactly one remote attempt, stale revision/fence rejection and read-only/held behavior from the loser. Fence loss after intent commit holds; it never authorizes another submission.

### 7. P0 — admission, reservation and wire amounts still disagree

The controller checks raw strategy volume while the adapter rounds volume to four decimals. Fresh execution reconfirmed an admitted wire notional larger than available balance.

**Required exit:** exchange metadata defines exact scales and rounding. Quantize once; derive immutable wire strings; use the same values for minimum, balance, caps, reservation, journal, transport and reconciliation. Mandatory below/exact/above matrices include `2 × 2.50006` with `5.00015` available and unequal asset decimals.

### 8. P1 — there is no maintained gate, fresh build result or external-semantics evidence

Both packages still lack tests; no CI exists; exact-SHA GitHub runs are absent. The fresh build attempt was blocked before execution and was not rerouted. Offline audit results are cache-dependent. No approved target account proves pagination, precision/minimums, fees, finality, request-reference behavior or timeout-after-acceptance recovery.

**Required exit:** one root command collects production-path tests under mandatory network denial and CI runs it on every push. Restore a fresh production build result under an approved command. Only after local P0 exits and explicit approval may a capped disposable account test external semantics; inability to recover ambiguous submissions leaves unattended trading unsupported.

## Architecture and implementation-plan refinements

- Selected the **durable-intent path** and made the protected instruction conflict a blocking gate rather than silently promising production.
- Added explicit admission states: `disabled`, `recovering`, `open`, `closing`, `held`, `stopped`.
- Added cross-process exclusive writer ownership/fencing.
- Split terminal `cancel-and-stop` from non-terminal `disable-and-hold`.
- Added exact file targets for Batches A-D and a required journal ADR.
- Added four executable acceptance matrices covering control/admission, crash/lifecycle, read/reconciliation and exact-money/accounting boundaries.
- Preserved live exchange work as a later, separately approved gate; local mocks cannot certify exchange semantics.

## Positive controls actually verified

- Listener binding remains `127.0.0.1`.
- Exchange transport remains centralized in `bot/dextrade.js`.
- Strategies remain I/O-free target generators.
- Scheduled ticks retain a non-overlap guard, though it does not serialize lifecycle or processes.
- All nine tracked bot JavaScript files parse.
- Direct installed production dependencies resolve.
- Runtime source and the real index matched the reviewed SHA before documentation edits; `vision.md` remained untracked and byte-identical.

These controls reduce exposure but do not establish production readiness.

## Reviewed source hashes

Git identities:

| Object | Hash |
|---|---|
| Source commit | `0040f42736f61dead2a612b4ed9295816ecd093a` |
| Real index/HEAD tree before review edits | `087093fafaede7fee40e8f5eaf6c263e35759cdb` |
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
| scratch safety probe | `a466e9dc57ea7db287106a3114f3795140048d175e46229f9665b147f74fe7db` |

The source hashes identify the exact implementation assessed even though documentation is now modified in the working tree. This review was not committed or pushed.
