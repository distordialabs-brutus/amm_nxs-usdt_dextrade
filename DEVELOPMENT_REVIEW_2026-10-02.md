# Development and Architecture Review — 2026-10-02

## Decision

> **Current source qualification:** this dated evidence remains scoped to `7fb784ac7f7f1d392751b9984f6be5917d5370c9`. The published documentation line now ends at remote tip `916aec034fccb7fbc19b2df0262c089ee0f14cc3`. Local-only `e0ad95cd7bb48dddd265b8771a0d72ebca8722b2` is a divergent sibling from the same merge base, not an ahead commit and not published evidence. Both retain unchanged runtime trees (`bot/` `cfe5e308bbc01fc5b55329bc4378ac449720a70d`, `src/` `eac7280ff4119b667d85fa2be66d3062fc35de58`). A later review found no maintained test script or CI and was approval-blocked from a composite fresh build/syntax/test probe; no new build pass is implied. Parse-only checks of `bot/index.js`, `bot/server.js`, and `bot/dextrade.js` passed, while both `npm test` invocations reported a missing script. Current architecture and repair order are in `ARCHITECTURE.md` and `DEVELOPMENT_PLAN.md`.

**Reviewed source and requested baseline:** `7fb784ac7f7f1d392751b9984f6be5917d5370c9` on `main`.

**Verdict: unsafe for unattended trading or meaningful capital.** `HEAD` equals the requested baseline, so there is no committed or working-tree runtime delta to accept after it. The only baseline-to-working-tree tracked changes at review start were documentation. Runtime source, manifests and lockfiles are unchanged; both packages still lack a test command; no CI workflow is tracked; and every P0 release gate remains open.

No live dex-trade or Nexus API request was made. The focused probe used synthetic credentials, replaced exchange/process boundaries and opened only one ephemeral `127.0.0.1` listener. It created or cancelled no exchange order and changed no account, credential, runtime source or dependency. The assessment phase did not stage, reset, clean, stash, commit or push the original checkout. Subsequent documentation-only publication used a detached worktree; the original checkout and its index were preserved.

## Baseline and preserved working tree

At review start:

- `HEAD` and baseline were `7fb784ac7f7f1d392751b9984f6be5917d5370c9`; `git log baseline..HEAD` was empty.
- Branch status was `## main...origin/main` with `0` ahead and `0` behind according to local tracking metadata.
- Modified tracked files were `ARCHITECTURE.md`, `DEVELOPMENT_PLAN.md` and `README.md`; there were no staged files.
- Untracked files were `DEVELOPMENT_REVIEW_2026-09-30.md` and `vision.md`.
- Initial real-index SHA-256 was `22ed12b0797d3b7fb924586ee5b211f198c266e8f4aca11491af3786d4ce73c8`.
- Initial staged-diff SHA-256 was the empty SHA-256 `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`.
- Initial unstaged-diff SHA-256 was `166edeee66aadf0f818a17e3694e391341220220ba260cc4fadaa7b1cb1f89b3`.
- Initial untracked hashes were `506a1399a0160f77deeb09160328f220d922d9bdff4cd0516138f9ddfdd10942` for `DEVELOPMENT_REVIEW_2026-09-30.md` and `733a95bbb8d59d0e55acebe486a3bbbca82b70f89d66bbe0a3b50c333685c5d0` for `vision.md`.
- Baseline trees were root `e64e6480d40152c3bcf4683362b79d8c512de4ec`, `bot/` `cfe5e308bbc01fc5b55329bc4378ac449720a70d` and `src/` `eac7280ff4119b667d85fa2be66d3062fc35de58`.

This review intentionally changes only `ARCHITECTURE.md`, `DEVELOPMENT_PLAN.md` and this dated evaluation. The pre-existing `README.md`, `DEVELOPMENT_REVIEW_2026-09-30.md` and `vision.md` remain outside this review's edit scope.

## Executed offline evidence

Toolchain: Node `v22.23.2`, npm `10.9.8`.

| Command / probe | Result |
|---|---|
| `npm run build -- --output-path /home/brutus/.hermes/profiles/principal-dev/cache/scratch/amm-nxs-usdt-review-2026-10-02/build` | **PASS** — webpack `5.99.9` compiled successfully in `766 ms`; `caniuse-lite` was reported 17 months old. No update was attempted. |
| `node --check` over the nine tracked files in `bot/*.js` and `bot/strategies/*.js` | **PASS** — `syntax_checked=9`. |
| Root `npm test` | **FAIL / absent gate** — exit `1`, `Missing script: "test"`. |
| Bot `npm test` | **FAIL / absent gate** — exit `1`, `Missing script: "test"`. |
| `git ls-files '.github/**' '*test*' '*spec*'` | Empty — no tracked test/spec/CI candidate. |
| Root `npm ls --depth=0 --omit=dev` | **PASS** — `nexus-module@1.1.11`, `react-redux@8.1.2`, `redux@4.1.2`. |
| Bot `npm ls --depth=0 --omit=dev` | **PASS** — `axios@1.13.5`, `cors@2.8.6`, `dotenv@16.6.1`, `express@4.22.1`. |
| Root and bot `npm audit --offline --omit=dev` | **PASS as cache-only checks** — each reported `found 0 vulnerabilities`; this is not current registry evidence. |
| Scratch scenarios `attacker-origin-control`, `import-side-effects`, `failed-balance-authority`, `negative-open-inference`, `missing-placement-identity`, `cancellation-cardinality`, `stop-placement-race`, `restart-state-loss`, `stop-without-cancel`, `exact-unit-mismatch` | **PASS as defect reproductions** — `10/10`, `0` harness failures. Passing means the unsafe behavior was reproduced, not that a safety gate passed. |
| Live exchange/Nexus semantics | **NOT RUN** by scope and repair order. |
| Exact-head publication CI | **ABSENT** — no tracked workflow; documentation publication is not a runtime acceptance gate. |

Scratch evidence:

- Probe: `/home/brutus/.hermes/profiles/principal-dev/cache/scratch/amm-nxs-usdt-review-2026-10-02/focused-lifecycle-probe.js`, SHA-256 `f75792eae27556448d51d0600929a5b6953205890b971da8423f3004174c538b`.
- Result: `/home/brutus/.hermes/profiles/principal-dev/cache/scratch/amm-nxs-usdt-review-2026-10-02/focused-lifecycle-result.json`, SHA-256 `4caf826548de25c77db5972a11f05a76fa5df6f68011168aef9235eb2254177d`.
- Bundle: `/home/brutus/.hermes/profiles/principal-dev/cache/scratch/amm-nxs-usdt-review-2026-10-02/build/app.js`, SHA-256 `821aba0d36f85b5e7306546edeb4e5313defb4235d8160c4a596e8db111b80dc`.
- License/map hashes: `025e0153dbe51a6d21bcbd90816c0ee531e8d960862280ea49d2298a93ec052c` and `8f48628b781335ea85c6d785cfd1eab0f2d2e0d5847627faab7fd3abe69d0f40`.
- Complete command/output and dirty-state manifest: `/home/brutus/.hermes/profiles/principal-dev/cache/scratch/amm-nxs-usdt-review-2026-10-02/evidence.md`.

## Fresh defect reproductions

1. **Attacker-origin control:** attacker-origin requests to `/api/start`, `/api/config`, `/api/rebalance` and `/api/stop` returned `200`, advertised `Access-Control-Allow-Origin: *` and invoked the controller. `numGrids: 1000000` and an unknown exposure key reached the controller.
2. **Import side effects:** importing `bot/index.js` attempted one listener, installed one interval and SIGINT/SIGTERM handlers, and initiated ticker, book and balance reads.
3. **Failed balance authority:** two failed balance refreshes left the seeded cached balance authoritative and two placements were attempted.
4. **Negative open-list inference:** an empty open-order response changed A/B to `filled` and fabricated buy volume `10`, sell volume `10`, buy cost `10`, sell revenue `12` and realized PnL `2` without fill evidence.
5. **Missing placement identity:** two `{}` placement responses made two remote attempts but collapsed local state to one literal `undefined` key.
6. **Cancellation cardinality:** one positive result for requested A/B/C caused all three local rows to become `cancelled`.
7. **Stop race:** stop reported `stopped` while placement awaited; accepted order `race-1` later became `open`, no cancellation occurred and `start()` installed one post-stop interval.
8. **Restart loss:** one managed order became zero after a fresh state-module load.
9. **Stop without cancel:** `stop(false)` reported `stopped`, attempted zero cancellations and retained A/B as open.
10. **Wire mismatch:** admission used raw `2 × 2.50006 = 5.00012` against `5.00015` USDT, while the real adapter emitted `2.0000 × 2.5001 = 5.0002`.

## Findings in financial-risk order

### 1. P0 — disabled trading and terminal stop are conflated

`stop(false)` and the deterministic placement race prove that closing local admission is not the same as reaching a terminal financial state. The current controller can report `stopped` while known or newly accepted orders remain open and while a timer can still be installed.

**Required exit:** expose independent `tradingAdmission`, `exposureState` and `lifecycleResult` fields. `disable-and-hold` returns `disabled_held`, exact reservations and `terminal: false`. Only a drained queue, zero future timers/private calls, positively terminal attributable intents and zero unresolved reservation may return `stopped_terminal` and `terminal: true`. Run the table in `DEVELOPMENT_PLAN.md` before and after process restart.

### 2. P0 — control and lifecycle authority are unauthenticated and unserialized

Loopback binding is not authorization. Wildcard CORS plus no capability allows all mutation routes to call the controller. Idle prefetch, timer ticks and route commands are coordinated by booleans rather than one generation-aware transition protocol.

**Required exit:** exact configured origin, non-bundled capability, closed schemas and limits must reject before controller invocation. One FIFO controller queue covers refresh, start, tick, config, rebalance, stop and shutdown, rechecking generation and writer fence after every await.

### 3. P0 — negative or stale evidence authorizes writes and fabricates terminal money state

Balance errors are logged and absorbed; cached balances can authorize placement. Open-order absence is treated as full fill even though the response has no completeness, pair or pagination proof. Book failure can fall back to last trade.

**Required exit:** every write consumes one fresh, pair-bound, schema-valid, complete evidence bundle. Any failed member produces a visible hold and zero dependent writes. Only positive closed/fill/cancellation evidence changes terminal state, reservations or PnL.

### 4. P0 — writes are not attributable, exact or restart-safe

The adapter generates millisecond references, rounds after controller checks, accepts missing identities and has no durable intent. Cancellation discards per-ID failures. Restart loses order and ambiguity state.

**Required exit:** quantize once; freeze exact wire strings and unique business identity; persist intent plus reservation before transport; mark `submitting` under current fence; submit once; persist identity or `outcome_unknown`; and reconcile from positive evidence. Cancellation needs one durable typed result per requested ID.

### 5. P0 — durable intent requires cross-process writer ownership

An in-process queue does not protect one account from two Node processes, a stale lease holder or a restored journal copy. Projection/settings snapshots must not overwrite newer financial transitions.

**Required exit:** acquire one OS-enforced lock or transactional lease before recovery or any private mutation. Every transition compares monotonic fence and expected revision. Two real processes must prove one owner and one remote attempt; stale owner, owner death, restored copy and fence loss after intent commit remain read-only/`disabled_held` with exact retained exposure.

### 6. P0 — repository instructions conflict with the selected safety design

`CLAUDE.md:29` still prohibits a database while restart-safe financial intent requires a transactional journal. This review did not modify the protected instruction or choose a storage driver.

**Required exit:** maintainer approval for the instruction correction and an ADR approving runtime/driver, durability, filesystems, migrations, lock/fence behavior, corruption refusal, backup/restore and packaging. Until then, durable-write work is blocked and trading stays disabled; no in-memory or JSON fallback is acceptable.

### 7. P1/P2 — no maintained gate or target-exchange evidence

The build and syntax checks pass, but no collected suite or CI exists. Offline audit output is cache-bound. No capped target account establishes pagination, precision/minimums, request-reference behavior, cancellation finality, fills/fees or timeout-after-acceptance recovery.

**Required exit:** implement the root `npm run verify` contract and Batches A-E offline, then separately approve capped Batch F target-account work. Mocks prove local containment only.

## Documentation changes and coding standard

`ARCHITECTURE.md` now makes the status model and coding boundary explicit: disabled admission, unresolved exposure and verified terminal stop are separate facts; financial transitions are exhaustive and typed; money is exact before admission; catch-and-log cannot grant authority; every await rechecks generation/revision/fence; and durable intents have one cross-process writer.

`DEVELOPMENT_PLAN.md` now requires one root verification command, zero-warning static checks, production-seam behavior tests, deterministic clocks/IDs, isolated journals, resource-leak checks, per-await fault barriers, exact-money boundary tests, exhaustive state transitions and actual two-process writer/fencing tests. It also defines the mandatory status table, including the current `stop(false)` case as a red test.

## Positive controls actually verified

- Listener binding remains `127.0.0.1`.
- Exchange transport remains centralized in `bot/dextrade.js`.
- Strategy modules remain I/O-free target generators.
- Scheduled trading ticks retain a non-overlap guard, though it does not serialize lifecycle commands or processes.
- The frontend production bundle builds and all nine tracked bot JavaScript files parse.
- Installed direct production dependencies resolve in both packages.

These controls do not establish production readiness.

## Reviewed implementation SHA-256

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

The hashes identify the unchanged implementation reviewed before these documentation edits. This dated evaluation records assessment-phase evidence; its publication changes documentation only and does not close any runtime acceptance gate.
