# NXS/USDT AMM Development Plan

This plan turns [`ARCHITECTURE.md`](ARCHITECTURE.md) into ordered, reviewable gates. Every implementation batch must keep the frontend build and bot syntax green, add collected regression coverage, deny unexpected network access, and preserve a default-disabled trading boundary.

## Current status — independent review 2026-10-02

Reviewed source HEAD and the requested baseline are both `7fb784ac7f7f1d392751b9984f6be5917d5370c9`; there are no commits or runtime changes after the baseline. At review start the existing working tree had modified `ARCHITECTURE.md`, `DEVELOPMENT_PLAN.md` and `README.md`, with untracked `DEVELOPMENT_REVIEW_2026-09-30.md` and `vision.md`. Runtime trees remain `bot/` `cfe5e308bbc01fc5b55329bc4378ac449720a70d` and `src/` `eac7280ff4119b667d85fa2be66d3062fc35de58`; no manifest, lockfile, test or workflow changed. Exact-head publication CI is outside this offline working-tree review; no tracked workflow exists to run.

Fresh offline execution passed the production frontend build, all nine tracked bot JavaScript syntax checks, both direct production dependency-tree checks and both cache-only audits. Both packages still lack a test script; no tests or `.github` workflow are tracked. A fresh ten-scenario production-path probe reconfirmed attacker-origin mutation, import side effects, placement after all balance refreshes failed, absence-based fabricated fills/PnL, missing-ID collapse, false cancellation finalization, in-memory restart loss, the stop/placement race, false `stop(false)` terminal state and raw/wire amount mismatch. Import still starts an unsynchronized idle prefetch that can overlap start. Every P0 remains open. See [`DEVELOPMENT_REVIEW_2026-10-02.md`](DEVELOPMENT_REVIEW_2026-10-02.md).

The untracked `vision.md` reviewed at SHA-256 `733a95bbb8d59d0e55acebe486a3bbbca82b70f89d66bbe0a3b50c333685c5d0` adds binding direction for this plan: preserve operator custody, make no underwriting claim, enforce a loss limit as well as order/exposure limits, fail closed under uncertain evidence, and publish inspectable release evidence. Treat it as context only until separately tracked by its owner; do not stage it with an implementation batch.

## Next coder handoff — start here, do not skip ahead

1. **Write the Batch A tests first with built-in `node:test`.** Add a root command equivalent to `node --require ./test/network-deny.js --test --test-concurrency=1 test/*.test.js`; the preload denies external DNS/socket/axios access while explicitly permitting only a test-owned ephemeral loopback listener. Tests import the real production modules and inject boundaries rather than replacing the controller under test.
2. **Make composition imports inert.** Add pure `parseConfig(env)`, injected `createController(deps)` and injected `createServer(deps)`. Export `buildRuntime(deps)` and `main()` from `bot/index.js`; only `require.main === module` may load dotenv, listen, install signals or schedule timers. Move idle prefetch behind the injected scheduler and generation check. Do not add placement/retry capability in this batch.
3. **Separate placement admission from emergency risk reduction.** Missing `TRADING_ENABLED`, origin, capability or any order/batch/inventory/unresolved/loss cap denies start, config that can widen risk, rebalance and placement with zero private calls. Authenticated `cancel-and-stop` closes placement first and may cancel only with a healthy journal/current writer fence and complete known exposure; otherwise it returns a quantified hold without an unjournaled private call.
4. **Then land Batch B as a separate acceptance-reviewed delta.** Add exact origin/capability parsing, closed request schemas and one generation-aware FIFO transition queue. Include idle prefetch, start, tick, config, rebalance, stop and shutdown. Plain `stopped` is forbidden while any placement/cancellation is accepted, open or unknown.
5. **Design typed reads and exact units before durable writes.** Change the exchange port so the adapter receives frozen `rateText`, `volumeText` and `requestId`; the adapter may not round numbers or generate a financial request ID. The first money test must reproduce and then reject the `2 × 2.50006` raw-versus-`2.5001` wire boundary from the current review.
6. **Resolve the repository instruction conflict before Batch D.** The durable-intent path is selected, while `CLAUDE.md` still prohibits any database. A maintainer must approve correction of that protected instruction and approve one audited transactional journal adapter. Until then, stop after containment/read work and keep financial writes disabled.

## Mandatory repair order

Do not reorder these batches to add trading capability before containment and its collected gate. Do not use live exchange credentials in Batches A-D.

### Definition of done for every batch

Every batch must leave one checked-in root command, `npm run verify`, that fails on the first broken gate and runs, in order: bot JavaScript syntax checks; lint/static checks with zero warnings; the complete `node:test` suite under the network-deny preload and `--test-concurrency=1`; and the production frontend build. CI calls only this command so local and publication gates cannot drift. Until this command exists, a build plus ad-hoc probes is evidence about the current snapshot, not an implementation acceptance gate.

Code review rejects a batch unless all of the following are true:

- Each changed financial invariant maps to one production module, one named collected test and one operator-visible failure code. No duplicated transition policy, mutable financial singleton, adapter-side money conversion or catch-and-log authority path is introduced.
- Public functions use closed request/result objects. Unknown keys, wrong scalar types, invalid enum members and missing required context fail before mutation. Tests assert stable codes and exact result fields rather than free-form message fragments.
- Tests invoke the production controller and real adapter serializer with only transport, clock, IDs, scheduler and storage-fault boundaries replaced. A mocked controller, helper-only reducer test or source-text assertion cannot close a behavioral exit.
- Every test uses a fake clock, deterministic IDs and its own temporary journal. Teardown proves no active listener, timer, signal handler, lock or temporary file remains. Tests pass alone, in shuffled file order and on a second immediate run.
- Every remote or storage `await` in changed code has a barrier case for stop, fence loss and thrown/malformed/ambiguous outcomes. Assertions include exact remote attempts, business identity, journal state/revision/fence, base/quote reservation, projection, timer count and restart result.
- Exact-money tests cover below/exact/above quantization, minimum, balance and every configured cap with unequal asset decimals. The adapter payload text must equal the frozen journal text byte-for-byte; no test computes its expected value with the production function under test.
- State-machine tests are table-driven over every legal and illegal transition. Unknown and duplicate evidence, stale revisions, stale fences and zero-entity reconciliation must fail closed. Branch or line coverage is diagnostic only; it does not replace the explicit transition matrix.
- Multi-process journal tests launch two actual Node processes against the same storage and include simultaneous start, stale owner, owner death, restored-copy and fence-loss-after-intent cases. Exactly one process may own writes and every loser remains read-only/`disabled_held`.
- Documentation, schema, status projection and tests change in the same batch when an externally visible state or error code changes. No readiness claim is made from mocks for pagination, cancellation finality, fills, fees or timeout-after-acceptance semantics.

Required review evidence is the exact source SHA/candidate diff hash, Node/npm versions, command and exit status, collected test names/count, requirement-to-test map, production file hashes and proof the real Git index plus unrelated dirty files were unchanged.

### Required status contract tests

Before any placement work, add a table-driven suite for the status split defined in `ARCHITECTURE.md`:

| Scenario | `tradingAdmission` | `exposureState` | `lifecycleResult` / `terminal` |
|---|---|---|---|
| Never enabled; no journal exposure | `disabled` | `clear` | `stopped_terminal` / `true` |
| `disable-and-hold` with known open order | `disabled` | `known_open` | `disabled_held` / `false` |
| Placement or cancellation outcome unknown | `disabled` | `outcome_unknown` | `disabled_held` / `false` |
| Enumeration incomplete, journal corrupt or fence lost | `disabled` | `inconsistent` | `disabled_held` / `false` |
| Cancel-and-stop with all attributable intents positively terminal and queue drained | `disabled` | `clear` | `stopped_terminal` / `true` |

For every row assert HTTP body, read projection, exact reserved units, journal state, remote attempts and timer count before and after a process restart. The current `stop(false)` reproduction is the mandatory red test: two retained open orders must never produce `stopped_terminal`.


### Batch A — import-safe seam, one gate, default deny (P0 containment)

**Files:** modify `bot/index.js`, `bot/server.js`, root `package.json`; create `bot/controller.js`, `bot/config.js`, `test/network-deny.js`, `test/import-safety.test.js`, `test/server.test.js`, and `.github/workflows/ci.yml`.

**Change:** reduce `bot/index.js` to guarded composition; add pure configuration parsing and an injected controller; make server construction import-safe and independent of singleton state; add one root `test` command and CI. Parse an independent `TRADING_ENABLED` switch as false unless explicitly enabled. Missing control policy or any per-order, batch, inventory, unresolved-exposure or loss policy keeps every risk-adding command disabled before it can reach a private exchange method. Status/liveness remain read-only. Risk-reducing stop follows the separate journal/fence/complete-exposure rule and otherwise returns held.

**Exit command:** one documented root `npm test` command runs every collected test from a clean install with real DNS/socket/axios access denied, then CI runs the same command on every push.

**Exit assertions:** importing controller, server, config and domain modules creates zero listeners, timers, signal handlers, credential reads or network calls; default configuration permits read-only liveness/status only; start/config/rebalance cause zero private calls when enablement, auth/origin or caps are missing. Re-run the 2026-10-02 import probe and require `0` for listener, interval, signal and exchange-read counts. Startup may explicitly construct the process shell, but idle prefetch must not outlive or race its admitted generation.

**Implementation sequence:**

1. Capture the current ten failures as named collected tests without changing production behavior.
2. Introduce `parseConfig(env)` and test missing, malformed, duplicate/ambiguous and complete policies as data only.
3. Change `createServer` to receive `readModel`, `authPolicy`, `logger` and `clock`; remove direct `state` reads from route code.
4. Extract controller logic from `bot/index.js` without changing exchange semantics. Inject exchange, projection, strategy registry, scheduler, clock and IDs.
5. Put listener/timer/signal creation behind `main()` and the `require.main` guard; then turn the import-safety assertions from red to green.
6. Add refusal-path HTTP tests and the external-network-deny preload; only then add CI. Batch A ends with writes default-disabled, not with a new working placement path.

### Batch B — authenticated control schemas and one lifecycle queue (P0 containment)

**Files:** modify `bot/controller.js`, `bot/server.js`, `src/App/Main.js`; create `bot/domain/admission.js`, `bot/domain/policy.js`, `test/controller-races.test.js`, and expand `test/server.test.js`.

**Change:** enforce exact configured origin plus a non-bundled control capability on every mutation route; install server-owned strategy schemas and hard per-order, batch, inventory, unresolved-outcome and loss caps. Route idle prefetch, scheduled tick, start, stop, config, forced rebalance and shutdown through one controller admission/transition protocol. Stop closes admission first, joins/resolves the active transition, reconciles attributable exposure, and prevents any later interval installation. Replace the boolean stop contract with a closed mode: terminal `cancel-and-stop`, or explicit `disable-and-hold` that reports held exposure and never claims `stopped`.

**Exit tests:** `test/server.test.js` rejects disabled, missing/wrong/duplicate capability, attacker/missing origin under browser policy, unknown top-level or strategy field, fractional count, `numGrids=1000000`, oversized body and over-cap notional with 4xx/503 and exactly zero controller, journal, state or exchange calls. It also proves the capability is absent from the production bundle and logs. Define and separately test the authenticated non-browser operator policy rather than implicitly trusting no `Origin` header.

`test/controller-races.test.js` places deterministic barriers before and after market, balance, open-order, journal, placement, cancellation and fill-history awaits. Record the session generation, admission state, remote-write count, timer count, journal state and quantified unresolved exposure at each barrier. Stop must linearize by changing admission to `closing`, invalidate the active generation, join the transition, and return terminal only after each attributable order is positively terminal. If placement acceptance cannot be resolved, stop returns a disabled held result; it never reports plain `stopped`, installs a timer, issues a subsequent private write or releases the reservation.

### Batch C — complete typed reads before any financial write (P0)

**Files:** modify `bot/dextrade.js` and `bot/controller.js`; create `bot/domain/evidence.js`, `bot/domain/exactMoney.js`, `test/read-evidence.test.js`, and `test/exact-money.test.js`.

**Change:** return typed evidence for symbol metadata, ticker/book, balances, open orders and fill/closed history: pair, fetched-at/freshness, validated schema, canonical IDs, precision/minimums, cursor/range and explicit completeness. Remove last-trade fallback for quoting and cached-balance authorization. Open-order absence never implies fill or cancellation.

**Exit tests:** `test/read-evidence.test.js` injects transport failure, malformed envelope/row, wrong pair, stale/crossed book, missing/duplicate/non-finite balance, non-monotonic/incomplete pagination and page-budget exhaustion through the actual controller. Every case records an operator-visible hold and causes zero cancellation, placement or PnL mutation. The captured empty-open-list case must leave both orders unresolved and PnL exactly unchanged. Balance failure before the tick and the second refresh inside `rebalance()` must each permit zero placements despite a populated cached snapshot; assert both read call count and zero private writes. A valid earlier snapshot cannot satisfy freshness for a later transition.

### Batch D — attributable writes, exact cancellation cardinality, durable ambiguity (P0)

**Prerequisite decision:** obtain maintainer approval to correct the protected `CLAUDE.md` in-memory-only rule and approve one audited transactional storage adapter. Create an architecture decision record at `docs/adr/0001-financial-intent-journal.md` naming the runtime/driver, durability settings, supported filesystems, lock/fencing model, backup/restore contract and rejection behavior. The 2026-09-28 host's experimental `node:sqlite` warning is not acceptance.

**Files:** modify `bot/controller.js`, `bot/dextrade.js`, `bot/config.js` and `bot/package.json`; create `bot/journal/index.js`, `bot/journal/schema.js`, `bot/journal/sqlite.js`, `bot/domain/orderLifecycle.js`, `test/journal.test.js`, `test/journal-process-worker.js`, `test/order-lifecycle.test.js`, and `test/cancellation.test.js`.

**Change:** implement the selected durable-intent journal path. Freeze pair, side, integer-scaled/decimal quantized price/volume, exact wire strings, notional, strategy/session generation and unique request reference before every minimum, balance, cap, reservation and submission decision. In one transaction create the intent and reservation; in a second fenced transaction mark `submitting` immediately before transport. A crash after `submitting` is ambiguous even if the call may not have reached the socket. Missing ID, malformed success, timeout, interruption or ID-persistence failure becomes `outcome_unknown`, aborts the batch and blocks replacement. Cancellation has its own persisted request transition and returns one typed result per requested ID: `confirmed_cancelled`, `still_open` or `outcome_unknown`. Acquire exclusive writer ownership/fencing before recovery or any private call; a second process is read-only/held. Every journal write compares the expected fence and revision; no cached projection/full-snapshot save may overwrite a newer financial transition.

**Exit tests:** `test/order-lifecycle.test.js` uses a durable temporary journal and separate-process restart fixtures. Fault-inject before intent, after intent/before call, after remote acceptance/before parse, after response/before identity persistence, after persistence/before finalization, during recovery and on duplicate invocation. For every row assert exact remote-write count, frozen terms/reference/wire strings, journal state, canonical exchange identity if known, reserved base/quote exposure and admission state. An accepted-but-unresolved action remains non-sendable after restart; recovery either attaches positive authoritative evidence without another submission or retains `outcome_unknown`/`manual_review`. A `{}` response makes exactly one call, creates no `undefined` key, stops the rest of the batch and remains durable across restart. The raw/wire boundary fixture must prove one exact admitted amount is identical in policy, journal, reservation, adapter payload and reconciliation. `test/journal.test.js` must also cover open/migration/write/sync/readback failure, late transaction rollback, corrupt/truncated storage, backup/restore, stale fence, two simultaneous processes and lock-owner death; every uncertain case keeps full exposure held and permits zero new submissions.

`test/cancellation.test.js` requires result cardinality and ID correspondence equal to the request. One success for A/B/C may finalize only A; B/C retain their orders and reservations. Timeout-after-cancel-acceptance remains unknown until direct evidence resolves it, and stop cannot downgrade that hold to terminal success.

### Batch E — startup adoption, exact fills and accounting (P0/P1)

**Change:** startup begins held and completely enumerates attributable open, closed, partial and unknown orders before first placement. Consume positive fill evidence with exact executed base/quote quantity, price, fee amount/asset and identity. Use explicit decimal or integer-scaled values, target symbol metadata and quantize before minimum/balance/cap checks.

**Exit tests:** crash/restart fixtures cover live owned, accepted-with-ID, accepted-with-lost-ID, unknown/unowned, partial, cancelled and filled orders without duplicate submission or fabricated terminal state. Startup performs zero placements until all journal rows are terminal or visibly held. Each recovery case asserts exact identity match, page completeness, remote-write count and retained exposure.

`test/accounting.test.js` feeds immutable fill IDs through the real reconciler: partial-fill-then-cancel, multiple fills at different prices, duplicate delivery, fee in NXS, fee in USDT, unequal asset decimals and rounding/minimum boundaries. Assert exact base/quote units, remaining reservation, inventory cost basis under one documented policy, realized/unrealized PnL and fee totals after every event. Submitted price/quantity must never substitute for missing fill fields. A deliberate duplicate or overpayment produces a positive reconciliation discrepancy; zero checked entities, malformed fill or incomplete history is unhealthy and leaves liabilities held.

### Batch F — target exchange semantics, dependencies and operations (P1/P2)

**Change:** only after A-E pass, use a capped disposable dex-trade sandbox/test account to establish pagination, pair filtering, symbol precision/minimums, fee fields, cancellation finality, idempotency/direct lookup and timeout-after-acceptance behavior. Upgrade dependencies only under the complete gate; document credentials, capabilities, held-state recovery, safe stop and incident response.

**Exit:** recorded capped test-account cases prove each external semantic that mocks cannot. `npm audit --omit=dev` meets the documented policy in both packages; build, collected tests and Nexus Wallet installation/polling/control smoke remain green. No production-readiness claim is allowed if the exchange lacks a proven recovery mechanism for ambiguous submissions.

## Explicit acceptance matrices

These matrices are mandatory requirements, not suggested examples. Every row must map to a named collected test that invokes the production controller/adapter path and asserts exact side-effect counts.

### A. Control and admission matrix

| Case | Expected HTTP/controller result | Journal/state/exchange effects |
|---|---|---|
| Trading switch absent/false | `503 disabled` | Zero controller construction capable of writes; zero timer/private call. |
| Origin absent on browser route, duplicated or not exact | `403` | Zero controller, journal or exchange call. |
| Capability absent, duplicated, malformed or wrong | `401/403` | Zero body-driven state mutation and zero controller call. |
| Unknown top-level/strategy key, fractional count, non-finite value or oversized body | `400/413` | Zero journal/state/exchange effect. |
| Any required order/batch/inventory/unresolved/loss cap absent | `503 disabled` | Read-only status only. |
| Over-cap proposal after exact quantization | Typed policy refusal | No intent and no remote call. |
| Journal unhealthy or writer fence unavailable | `held` | Full known exposure retained; zero new submission/cancellation. |
| Valid policy/auth/schema/fresh evidence with writer fence | Admitted once | Exactly one transition enters the journal protocol. |

### B. Lifecycle, journal and crash matrix

| Injected boundary | Durable state after recovery | Remote write count | Admission/result |
|---|---|---:|---|
| Before intent transaction commits | No intent/reservation | 0 | Refused; retry may create a new intent. |
| After committed intent, before `submitting` | `prepared`, exact reservation | 0 | Recoverable only through the same intent/reference. |
| After durable `submitting`, before transport invocation | `submitting` | 0 or unknown; never infer 0 after crash | Held until protocol-specific evidence proves no acceptance. |
| After remote acceptance, before response parse | `outcome_unknown` on restart | 1 | Held; no automatic resubmit/replacement. |
| After response parse, before exchange-ID commit | `outcome_unknown`, frozen response diagnostic | 1 | Resolve by authoritative identity/evidence only. |
| After exchange-ID commit, before projection update | Accepted identity retained | 1 | Rebuild projection; no resubmit. |
| Cancellation timeout after possible acceptance | `cancel_requested/outcome_unknown` | 1 cancel attempt | Exposure retained; terminal stop forbidden. |
| Stop races any await | Generation invalidated, active intent retained | At most admitted call | `held` unless every order becomes positively terminal; no later timer/private call. |
| `disable-and-hold` with open orders | Open identities/reservations retained | 0 cancellation if explicitly selected | State is `held`, never `stopped`. |
| Two processes share one journal | One current fence owner | Exactly one submission across both | Loser read-only/held; stale writes rejected. |
| Journal open/write/sync/readback/lock failure | Prior committed evidence preserved or corruption hold | 0 new writes | Held and operator-visible. |
| Restart with non-terminal rows | Rows enumerated before strategy admission | 0 new placement until complete | `recovering` then `open` or `held`. |

### C. Read and reconciliation matrix

| Evidence case | Required result | Forbidden effects |
|---|---|---|
| Transport error, malformed envelope/row or wrong pair | Typed incomplete/invalid hold | Cancellation, placement, PnL mutation, cursor advance. |
| Stale ticker/book/balance snapshot | Hold | Cached display values authorizing a write. |
| Empty, inverted or crossed book | Hold | Last-trade fallback for quoting. |
| Missing/duplicate/non-integer-scaled balance | Hold | Retaining an older balance as authority. |
| Non-monotonic pages, duplicate cursor or page-budget exhaustion | Incomplete hold with cursor evidence | Treating a partial scan as complete. |
| Managed order absent from complete open page only | Remains unresolved | Fill/cancel transition or reservation release. |
| Positive partial fills with complete history | Apply each immutable fill once | Submitted quantity/price substituted for execution evidence. |
| Zero checked expected entities | Unhealthy/incomplete | Green reconciliation. |

### D. Exact-money and accounting matrix

| Boundary | Assertions |
|---|---|
| Price/volume below, exactly at and above one quantization unit | One frozen integer/exact-decimal representation produces policy values, journal fields, reservation and wire strings; no adapter re-rounding. |
| `price=2`, proposal volume `2.50006`, available quote `5.00015` under four-decimal volume | Either freeze `2.5001` and reject notional `5.0002`, or apply the exchange's proven rounding policy consistently; never admit raw `5.00012` then transmit more. |
| Below/exact/above exchange minimum | Decision uses frozen wire notional and exchange metadata. |
| Unequal NXS/USDT decimal scales | Exact base/quote units and formatted public values match their own scales. |
| Partial fill then cancel | Executed units and fees post once; only positively cancelled remainder releases. |
| Duplicate fill ID | No second inventory/PnL/fee mutation. |
| Fee in NXS vs fee in USDT | Correct asset-specific units, inventory and PnL adjustment. |
| Malformed/incomplete fill history | Full unresolved reservation remains; reconciliation unhealthy. |

## Acceptance evidence format

Each batch review must publish the exact source SHA, command, collected test names/count, and a requirement-to-test/code map. For financial transitions, include a compact matrix with: injected boundary; local state before/after; journal state; remote attempts and identities; exact base/quote reservation; admission/held state; and restart result. Assertions that accept any thrown error, helper-only tests that bypass the controller, or mocks that pre-decide completeness do not satisfy an exit.

Keep local containment, target-account external semantics and production operation as separate verdicts. No live exchange request is permitted before A-E pass and an explicit capped Batch F exercise is approved. Offline `npm audit` output is informational only; it cannot close a prior online advisory finding.

## Release-gate status

| Priority | Gate | Executable exit criterion | Status |
|---|---|---|---|
| P0 | Default-deny containment | Missing enable/auth/origin/order/exposure/loss configuration permits zero private exchange writes | **Not implemented** |
| P0 | Test/CI contract | One network-denying collected command covers controller, adapter, strategy and HTTP behavior on every push | **Not implemented** |
| P0 | Durable-journal authority | Maintainer corrects protected in-memory-only instruction and approves one transactional adapter/ADR | **Blocked on approval; not implemented** |
| P0 | Control authorization/input limits | Untrusted origin/credential and out-of-schema/over-cap requests are rejected with zero effects | **Not implemented** |
| P0 | Serialized lifecycle/safe stop | Await barriers and `disable-and-hold` cannot produce a false terminal stop; ambiguity returns a quantified held result | **Not implemented; stop race and no-cancel failures executed 2026-10-02** |
| P0 | Fresh complete read evidence | Failed/malformed/stale/incomplete reads cause a visible hold and zero writes/PnL | **Not implemented** |
| P0 | Durable attributable writes | Ambiguous writes survive restart, retain exposure and prohibit later-batch/retry/replacement | **Not implemented** |
| P0 | Cross-process writer fencing | Two processes sharing one journal produce one fence owner and at most one remote submission | **Not implemented** |
| P0 | Exact cancellation cardinality | Every requested ID has a typed result; only positive evidence finalizes it | **Not implemented** |
| P0 | Restart reconciliation | Startup adopts or holds all attributable/unknown exposure before placement | **Not implemented** |
| P0 | Exact wire-unit admission | Policy, reservation, journal and API payload use one quantized price/quantity/notional | **Not implemented; mismatch executed 2026-09-28** |
| P1 | Exact fill/fee accounting | Fill-level quantities, prices, decimals and fees produce deterministic exact results | **Not implemented** |
| P1 | Live exchange semantics | Capped sandbox/test-account cases prove pagination, finality and ambiguity recovery | **Not implemented** |
| P2 | Dependency remediation | Audits meet policy with build, tests and wallet compatibility green | **Deferred / compatibility-gated** |
| P2 | Operational runbook | Credentials, control capability, holds, stop and incident response are exercised | **Partial** |

**Release decision:** do not use meaningful capital or unattended operation until every P0 item passes, P1 money behavior receives independent review, and capped live-boundary evidence is complete.
