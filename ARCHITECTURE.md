# NXS/USDT AMM Architecture

## Governing vision and portfolio traceability

Read [the repository vision](vision.md) and [Distordia alignment/dependency map](docs/DISTORDIA_ALIGNMENT.md) before assigning work. Authority is master Distordia strategy/customer evidence → portfolio roadmap/strategy decisions → repository vision → this architecture → tasks/code/tests/external evidence and human release.

**Portfolio purpose:** O4 bounded operator risk and reconcilable exchange action; optional ecosystem-enabling experiment. Operator-controlled exchange accounts and explicitly limited authority; truthful held exposure and exact fills/fees. Not an on-chain AMM, a pooled treasury or Distordia underwriting. The alignment map supplies customer-evidence qualification, batch ownership, upstream prerequisites and human gates. Each material task must name those fields alongside its exact production paths and collected acceptance tests. This documentation alignment changes no runtime, test result or release status; dated evidence below remains evidence for its stated snapshot only.

## Purpose and current boundary

This repository contains two independently packaged processes:

```text
Nexus Wallet module (React/Redux)
  -- HTTP polling every 4 seconds -->
Loopback Express control/status API (:17442)
  --> in-memory bot state + guarded 15-second tick
  --> rate-limited dex-trade REST client
  --> dex-trade.com NXS/USDT order book
```

The frontend is an operator console, not the source of trading truth. The bot owns strategy selection, order placement, cancellation, reconciliation and the in-memory status snapshot. NXS is traded as an exchange asset here; this implementation does not submit transactions through the Nexus node API.

### Current source snapshot — 2026-10-02

Reviewed source HEAD and the requested baseline are both `7fb784ac7f7f1d392751b9984f6be5917d5370c9` on `main`; there are no commits or runtime changes after the baseline. At review start the working tree already contained modified `ARCHITECTURE.md`, `DEVELOPMENT_PLAN.md` and `README.md`, plus untracked `DEVELOPMENT_REVIEW_2026-09-30.md` and `vision.md`. `bot/` remains tree `cfe5e308bbc01fc5b55329bc4378ac449720a70d` and `src/` remains `eac7280ff4119b667d85fa2be66d3062fc35de58`; no runtime, manifest, lockfile, test or workflow repair landed.

The untracked `vision.md` reviewed at SHA-256 `733a95bbb8d59d0e55acebe486a3bbbca82b70f89d66bbe0a3b50c333685c5d0` is context, not part of source HEAD. Its operator-custody, no-underwriting, bounded-authority, exact-money, reconciliation-first and transparent-evidence doctrine is adopted here as a design constraint. That does not make the untracked file publishable or implemented. Every P0 release blocker remains open; fresh build, syntax, dependency and ten-scenario defect-reproduction evidence is in [`DEVELOPMENT_REVIEW_2026-10-02.md`](DEVELOPMENT_REVIEW_2026-10-02.md).

`bot/index.js` binds the API to `127.0.0.1`, which reduces network exposure but is not authorization. `bot/server.js` currently returns `Access-Control-Allow-Origin: *` and has no control token. The server therefore grants arbitrary origins access to start, stop, configuration and rebalance routes; actual webpage reachability also depends on the browser's private-network and mixed-content policies, which were not exercised. Do not rely on those browser policies as server authorization. The loopback boundary must not be widened, and mutating routes require an authenticated, origin-restricted control boundary before release.

## Components

- `src/App/Main.js`: dashboard, polling and operator commands.
- `bot/server.js`: local status/control API and basic input-shape validation.
- `bot/index.js`: lifecycle, non-overlapping tick, market/balance reads, reconciliation and rebalance execution. Importing it also starts the listener, which currently prevents isolated controller tests.
- `bot/state.js`: ephemeral process state; restart loses managed-order and PnL history.
- `bot/strategies/`: pure target-order generation behind a common strategy contract.
- `bot/dextrade.js`: all exchange transport, signing and rate limiting.

## Binding durability decision and instruction conflict

The durable-intent path is selected. Unattended or restartable financial writes require a dedicated transactional safety journal for immutable placement/cancellation intent, exact quantized terms and reservations, remote identities, outcome-unknown state, recovery evidence and operator disposition. Ordinary market/status/UI projections may remain in memory; the safety journal is not optional application state and may not be replaced by process memory or an ad-hoc JSON snapshot.

This decision conflicts with the current `CLAUDE.md` statement that all bot state must remain in memory and no database may be added. For financial state, that statement is superseded by this architecture and [`ARCHITECTURE_ADDENDUM_2026-09-12.md`](ARCHITECTURE_ADDENDUM_2026-09-12.md). The attempted documentation correction was blocked by protected-file approval during the 2026-09-28 review, so implementation must treat the conflict itself as a gate: a maintainer must approve the instruction correction and an audited transactional adapter before Batch D begins. If either approval is absent, trading stays default-disabled; the fallback is not an in-memory journal and not a production-readiness claim.

The storage engine is deliberately not certified by this documentation review. The review host's Node `v22.23.2` exposes `node:sqlite`, but emits an `ExperimentalWarning`; host availability is not a production storage contract. Batch D must select and review one adapter, prove atomic commit/rollback, durable restart behavior, corruption handling, backup/restore and packaging on every supported runtime, and record the decision. Failure of the journal to open, migrate, lock, write, sync or reread closes write admission.

## Target clean architecture

Keep policy independent of Express, timers, environment variables and axios:

```text
process composition shell (env, listener, timers, signals)
  -> HTTP and scheduler adapters
  -> application controller / one serialized transition queue
  -> pure order-lifecycle, exposure and exact-money domain
  -> ports: exchange evidence + durable intent journal + clock/ID source
  <- dex-trade and journal adapters implement those ports
  -> read-only status projection
```

Dependencies point inward. `bot/index.js` must become a composition shell rather than owning process startup, controller policy, exchange calls and mutable financial transitions in one module. Domain transitions accept typed evidence and return explicit effects; only adapters perform I/O. The controller is the sole writer of lifecycle state and exposure reservations. The dashboard consumes projections and cannot authorize or infer financial outcomes.

Construction must inject the exchange, journal, clock, request-ID source and scheduler. Importing controller, server or domain modules must not open a listener, install signal handlers, load production credentials or reach the network. Keep strategy calculations pure, but validate their outputs and all exposure policy at the application boundary before creating an intent.

Fresh 2026-10-02 isolated evidence shows why this seam is the first architecture gate. With HTTP and exchange modules intercepted, importing the real `bot/index.js` still attempted the loopback listener, one interval, SIGINT/SIGTERM handlers and initial ticker, order-book and balance reads. The idle prefetch starts outside the trading lifecycle queue; if start wins after its entry check, the prefetch can continue mutating market/balance projection concurrently with the first trading transition. An ephemeral loopback server built from the real `bot/server.js` forwarded all four attacker-origin mutation routes to a fake controller without authentication. A deterministic await barrier again proved the placement race: stop observed no managed order while placement was pending, returned `stopped`, the accepted placement then became an uncancelled local `open` order, and `start()` installed a new trading interval after stop. These are production-path probes with external boundaries replaced, not a checked-in test suite.

The first implementation slice is deliberately narrow. Reduce `bot/index.js` to composition, move application transitions into `bot/controller.js`, centralize parsed fail-closed policy in `bot/config.js`, and keep route authorization/schema enforcement in `bot/server.js`. `src/App/Main.js` must send the configured non-bundled capability without becoming an authority source. `test/import-safety.test.js` and `test/server.test.js` must prove zero import side effects and zero controller calls for disabled, unauthenticated, wrong-origin, unknown-field or over-cap requests before order-lifecycle work begins.

### Required module contracts for the first implementation slices

These are coding contracts, not naming suggestions:

- `bot/index.js` exports `buildRuntime(deps)` and `main()` and calls `main()` only under `require.main === module`. It is the only module allowed to load dotenv, create the HTTP listener, install process signals or own process timers. `buildRuntime()` receives an already parsed configuration in tests; importing the file performs no work.
- `bot/config.js` exports a pure `parseConfig(env)` that returns an immutable `{ control, risk, exchange, runtime }` object plus explicit validation errors. Credential presence never implies `TRADING_ENABLED`. Missing or invalid write policy yields `placementEnabled: false`; it does not silently fill a permissive default.
- `bot/server.js` exports `createServer({ controller, readModel, authPolicy, logger, clock })`. It must not import mutable singleton state. Authorization, exact-origin handling, body limits and closed command schemas run before a controller method. A rejected request has zero controller calls.
- `bot/controller.js` exports `createController({ exchange, journal, projection, policy, scheduler, clock, ids })`. Every lifecycle command and timer callback enters one FIFO transition queue carrying an admission generation and writer fence. No method exposes the exchange adapter or mutable state directly.
- `bot/dextrade.js` accepts already frozen wire values and request references: placement receives `{ pair, side, rateText, volumeText, requestId }`; cancellation receives `{ exchangeOrderId, requestId }`. It must reject numeric price/volume inputs, must not call `toFixed()` or `Date.now()` for a financial write, and must return typed evidence rather than `data || data` shape guessing.
- `bot/state.js` becomes a read projection only. Journal rows and exposure reservations are not written through a full-state snapshot. Projection update failure after a committed journal transition is rebuildable and cannot reverse authority.
- The future journal port owns `acquireWriter`, `beginRecovery`, `preparePlacement`, `markSubmitting`, `recordAcceptance`, `markOutcomeUnknown`, `prepareCancellation`, `recordCancellationEvidence`, `recordFillEvidence` and `recordDisposition`. Every mutating call carries the expected monotonic fence/revision and fails closed on mismatch.

Risk-adding and risk-reducing writes have separate admission. Missing placement enablement or a risk cap forbids start, rebalance and new orders. An authenticated `cancel-and-stop` may attempt a known risk-reducing cancellation only while the journal is healthy, writer ownership is current and the complete attributable order set is retained. If those prerequisites fail, local placement admission still closes, but the result is `held`; the controller must not issue an unjournaled cancellation or claim terminal stop.

### Disabled trading is not terminal stop

The public status model must carry three independent facts instead of overloading one string:

- `tradingAdmission`: `disabled | recovering | open | closing` describes whether a new risk-adding transition may start.
- `exposureState`: `clear | known_open | outcome_unknown | inconsistent` describes retained exchange exposure and reconciliation confidence.
- `lifecycleResult`: `active | disabled_held | stopped_terminal` describes the operator-visible outcome.

`disabled_held` is mandatory whenever trading admission is closed but any attributable order is open, partially filled, cancel-pending, outcome-unknown, incompletely enumerated or manually held. Its projection includes exact reserved NXS/USDT units, counts by lifecycle state, the oldest unresolved intent and the evidence reason. `stopped_terminal` is legal only when the transition queue is drained, no timer can be installed, current writer ownership is released or intentionally retained read-only, every attributable intent is positively terminal, and unresolved reservation is exactly zero. A boolean such as `cancelOrders: false` may never map to `stopped_terminal`.

The HTTP contract uses closed commands (`cancel-and-stop`, `disable-and-hold`) and returns the three fields above plus `terminal: boolean`. Tests must reject unknown modes and must assert the response body, projection, journal rows, reservation totals, remote-call count and timer count. A successful HTTP status alone is not acceptance.

### Coding-quality contract

Financial code must be simple enough to audit and strict enough to fail closed:

1. Domain transitions are pure, exhaustive functions over explicit lifecycle states and typed evidence. Unknown states, fields or adapter outcomes return a coded refusal; no default branch converts them to success.
2. The controller is the only in-process financial writer. Modules export factories rather than mutable singletons; dependencies, clock, IDs, scheduler and logger are injected. No constructor or import performs I/O.
3. Remote adapters validate one documented response envelope and exact field types at their boundary. They return tagged results such as `accepted`, `rejected_before_acceptance`, `outcome_unknown` and `incomplete`; callers do not inspect arbitrary response shapes.
4. Catch-and-log is forbidden on an authority path. A read, journal, fence or schema failure must propagate as a typed hold and prohibit dependent writes. Logs are structured, correlation-ID-bearing and credential/capability-redacted.
5. Exact price, quantity, notional, fees and reservations have one immutable integer-scaled or exact-decimal representation. Conversion to wire text happens once before admission; JavaScript `number`, truthiness fallbacks and adapter-side rounding are forbidden for money authority.
6. Every intent, submission, cancellation, fill, disposition and projection rebuild is idempotent under a stable business identity. Journal transitions compare both expected revision and current writer fence.
7. The journal has one cross-process owner before recovery or mutation. Use an OS lock or transactional lease with monotonic fencing and explicit owner death/stale lease behavior. Settings and projection writers use separate records and cannot overwrite journal state through a full-object save.
8. Every `await` is treated as a cancellation and fence boundary. On resume, code rechecks generation, admission, journal revision and writer ownership before the next local or remote effect.
9. Functions remain narrow: policy does not call transport, transport does not decide exposure, projections do not grant authority, and HTTP status does not infer financial terminality. Duplicated lifecycle rules block review.
10. New behavior is incomplete without deterministic collected tests that reach the production controller/adapter seam, deny unplanned network access and assert exact side-effect counts. Source-shape or helper-only tests may supplement but never replace behavior tests.

The maintained root verification command must run syntax/static checks, lint with zero warnings, the complete serial safety suite and the production build. Tests use fake clocks and deterministic IDs, leave no listeners/timers/files behind, and never depend on order, wall time, credentials or the public network. Each failure injection asserts coded result, journal state/revision/fence, exact reservation, remote identities/attempts, projection and restart behavior; an assertion that accepts any exception does not satisfy the contract.

## Money and authority model

- Assets are NXS inventory and USDT quote currency. Amounts and prices are currently JavaScript floating-point numbers; orders are formatted to four decimal places.
- dex-trade is authoritative for order acceptance, cancellation, open/closed state, fills, execution quantities/prices, fees and account balances.
- `state.managedOrders` and `state.pnl` are process-local projections, not authoritative records. They are lost on restart.
- Every open or outcome-unknown order remains potential inventory exposure until exact exchange evidence resolves it. A timeout, malformed response, failed scan or bounded absence is not a terminal outcome.
- Private request references are currently millisecond timestamps created immediately before each call. They are not durable intents, are not serialized for uniqueness, and have no locally proven idempotency or direct-recovery contract.

## Invariants

1. At most one scheduled trading tick executes at a time (`tickInProgress`) — **implemented for ticks only**.
2. Only the dex-trade adapter communicates with the exchange — **implemented**.
3. A strategy computes targets but never performs I/O or mutates shared state — **implemented**.
4. Every placement batch uses a fresh, schema-valid balance and market snapshot and exchange-provided precision/minimum metadata — **not implemented**; balance and order-book failures can be absorbed and precision/minimums are hard-coded.
5. An order is terminal only from authoritative complete, paginated exchange evidence; absence from an open-order list is not proof of a full fill or cancellation — **not implemented**.
6. Realized PnL uses authoritative fill quantity, execution price and fees, not submitted-order values — **not implemented**.
7. Cancellation and placement have durable pre-submit intent, per-order identities and explicit `submitted`, `confirmed`, `outcome_unknown` and held transitions; ambiguous outcomes prohibit replacement risk — **not implemented**.
8. Control-plane requests cannot bypass exchange serialization or race cancellation/rebalance transitions — **not implemented**.
9. Mutating HTTP routes authenticate the intended wallet module and reject untrusted origins and out-of-schema strategy values — **not implemented**.
10. Private signing follows the exchange contract in `bot/dextrade.js`: SHA-256 over recursively sorted body values followed by the secret. It is not HMAC — **implemented and fixture-probed locally**.
11. Price, volume, minimum, balance, reservation, exposure and loss-limit checks use the exact same quantized wire units — **not implemented**; the controller checks raw floating-point volume but the adapter later rounds it to four decimals.
12. Operator policy caps per-order, batch, inventory, unresolved exposure and realized/unrealized loss without pooling capital or implying a guarantee — **not implemented**.

## Evidence-gated order lifecycle

The required lifecycle is:

```text
validate control request and fresh read evidence
  -> durably persist placement intent and immutable request reference
  -> submit once with an attributable request identity
  -> record canonical exchange order identity
  -> resolve ambiguous outcomes from authoritative exchange evidence
  -> record exact fills/cancellation and fees
  -> permit replacement only after exposure is resolved
```

Cancellation must return one result per requested ID: `confirmed_cancelled`, `still_open`, or `outcome_unknown`. Rebalance and stop may not mark all requested orders cancelled from a best-effort batch result. A placement timeout or missing ID enters `outcome_unknown`, aborts the remaining placement batch and reserves the unresolved exposure; it never permits blind resubmission. The controller, not logging in the adapter, owns the hold gate.

### Durable intent and recovery contract

Until target-account evidence proves a stronger exchange-native idempotency and direct-lookup contract, the safe baseline is a durable journal behind the injected journal port. Each placement/cancellation record must freeze a generated intent ID, exchange pair, side, integer-scaled/decimal price and quantity, quantized notional, strategy/session generation, client request reference, creation time and current state. Exchange IDs, exact response evidence and resolution attempts append to that identity; they never replace it. Required states are at least `prepared`, `submitting`, `accepted`, `outcome_unknown`, `partially_filled`, `filled`, `cancel_requested`, `cancelled`, `rejected` and `manual_review`. Only `filled`, positively confirmed `cancelled`, or authoritative `rejected-before-acceptance` releases all unresolved exposure.

The journal write that reserves exposure must commit before transport. A transport timeout, malformed success, process interruption or failure to persist the returned exchange ID moves or recovers the intent as `outcome_unknown`; it is not a retryable failure. Recovery starts with trading admission closed, enumerates every non-terminal journal row, performs exact identity lookup or complete attributable open/closed/fill enumeration, persists the evidence, and only then enables new placement. If the exchange cannot look up the durable client reference and completeness cannot be proven, the intent remains held for explicit operator disposition.

Crash/restart acceptance must cover every boundary: before intent commit; after intent/before transport; after remote acceptance/before response parsing; after response/before exchange-ID persistence; after identity persistence/before lifecycle finalization; during cancellation; and during restart recovery. For every fixture, assert the number of remote write attempts, the exact retained exposure, the recovered identity/evidence and whether admission is open. The safe result is one attributable remote action or a non-sendable hold—never an automatic second action inferred from local absence.

Read adapters must reject malformed envelopes, malformed records and incomplete pagination. `getOpenOrders()` currently has no pagination arguments or completeness result, while reconciliation treats its returned set as complete. Open-order transport/schema/page-budget failure must abort the trading transition, not merely skip reconciliation and continue to place. Missing orders require exact closed-order/fill evidence. A failed balance refresh cannot authorize cached balances. Missing, inverted or stale order-book evidence must not silently substitute last-trade price for a market-making placement decision.

Ordinary dashboard state can remain in memory under the existing no-database constraint, but an in-memory placement intent cannot survive a crash after remote acceptance. Before unattended trading, the architecture must either approve a minimal durable intent journal or prove that the exchange provides a unique idempotent client reference and direct authoritative lookup sufficient to recover every ambiguous submission. Startup remains held until that evidence reconstructs attributable live orders and unresolved exposure, or an operator explicitly disposes it. Without one of those recovery mechanisms, unattended restart is unsupported.

## Control and concurrency boundary

The scheduled tick guard does not serialize HTTP stop/config/rebalance calls with a tick already in flight. The 2026-10-02 isolated barrier probe executed the concrete failure: `start()` was paused inside `createLimitOrder()`, `stop(true)` found no managed ID and returned `stopped`, then the accepted order was recorded `open`; after the original tick returned, `start()` installed another interval. A single controller transition queue must cover idle prefetch, start, stop, configuration, forced rebalance, reconciliation, cancellation and placement. Stop must close admission, join or resolve an in-flight transition, reconcile attributable exposure, and only then report terminal state. The composition shell must not install a timer after admission has closed.

The queue requires explicit linearization semantics, not merely a mutex around route handlers. Every start creates a session generation. Stop first atomically changes admission from `open` to `closing`, invalidates that generation and cancels future timer admission. It then joins the active transition. After every awaited exchange call, the transition must re-check its generation before any further write. If a placement may have been accepted, stop must persist/recover its identity and positively cancel or classify it as a visible unresolved hold. Terminal `stopped` means no admitted transition can install a timer or issue another private call and every attributable order is terminal; otherwise the truthful result is a stopped/disabled **held** state with quantified unresolved exposure, not success.

The queue serializes only one controller process. Before any private call, the journal must also establish one active writer through an OS-enforced exclusive lock or a transactional lease with fencing token. A second process, stale lease holder, restored copy or independently launched bot must be unable to submit, cancel, advance recovery or overwrite disposition under an old fence. Loss of ownership after intent commit changes admission to held; it never authorizes a second submission. Multi-process tests must start two real Node processes against the same journal and prove one writer, one remote attempt, monotonic revision/fence checks and read-only status from the loser.

`POST /api/stop` currently accepts `cancelOrders: false`. Fresh 2026-10-02 execution placed two orders, skipped cancellation, returned `status: stopped`, and retained both local rows as `open`. The target protocol may offer **disable-and-hold-open-orders** as a separate explicit command, but it may not call that state terminal `stopped`. A terminal stop always closes admission, resolves or holds every attributable order, and truthfully reports residual exposure. The API must reject ambiguous booleans and unknown stop modes through a closed schema.

### Admission-state contract

| State | New private writes | Read-only refresh | Required transition evidence |
|---|---:|---:|---|
| `disabled` | No | Optional | Complete policy and explicit operator enablement are absent or false. |
| `recovering` | Recovery lookups only | Yes | Journal opened and exclusively owned; every non-terminal intent is being enumerated. |
| `open` | Yes, through one admitted transition | Yes | Policy, writer fence, journal health, fresh complete exchange evidence and exposure capacity are all valid. |
| `closing` | No new work; resolution/cancellation only | Yes | Generation invalidated; active transition joined; outstanding intents retained. |
| `held` | No | Yes | One or more quantified unresolved outcomes, incomplete reads, journal faults, fence loss or policy breaches remain. |
| `stopped` | No | Yes | No admitted work/timer can restart, and every attributable order is positively terminal with zero unresolved reservation. |

No route or scheduler callback may jump directly from `disabled`, `recovering`, `closing` or `held` to a financial write. Only a successful, journaled transition into `open` permits submission.

HTTP authorization is a separate adapter concern. CORS headers are browser response policy, not authentication. Mutation must be default-disabled, require an exact configured origin and a high-entropy capability supplied at runtime (never bundled in `dist/app.js`), and reject a missing/duplicate/malformed credential before parsing or invoking controller work. Define an explicit non-browser operator path rather than treating absent `Origin` as trusted. Read routes must expose only the intended projection; logs and strategy/status fields should be reviewed for credential or sensitive exchange-data leakage. Rate limits and body-size limits supplement but do not replace authorization.

Frontend `min`, `max` and `step` fields are presentation hints only. The server must own allowlisted schemas, reject unknown top-level and strategy fields, values outside each strategy's schema, non-integer count fields and explicit per-order/batch/inventory/unresolved-exposure ceilings before mutating state. Validation and authorization failures must produce exactly zero controller, journal, state or exchange calls. The currently accepted finite-only values permit configurations such as `numGrids=1000000`.

## Authoritative reads, fills and accounting

Every write decision consumes one validated evidence bundle: pair identity, fetch time/age, complete page/range proof, canonical order IDs, symbol precision/minimum metadata, balances and a non-crossed book. Failure of any required member invalidates the whole bundle. Cached values remain display-only. An open-order list is positive evidence for rows it contains; absence is never fill/cancellation evidence without a complete authoritative closed-order/fill lookup.

Quantization is a domain boundary, not an adapter formatting detail. Convert strategy proposals into integer base/quote units or an exact decimal representation using fetched symbol metadata, then derive the exact wire strings once. Minimum, available-balance, per-order/batch/inventory/unresolved-exposure and loss-limit checks, reservations, journal rows, API submission and reconciliation must all consume those frozen units. Fresh execution demonstrated the current mismatch: raw `price=2`, `volume=2.50006` passes a `5.00015` USDT balance check at raw notional `5.00012`, while the real adapter emits volume `2.5001`, a wire notional of `5.0002`. The journal may never retain a raw amount different from the submitted amount.

Accounting consumes immutable fill events keyed by exchange order and fill/trade identity. Each event carries executed base quantity, quote quantity or execution price, fee amount and fee asset, side and authoritative timestamp. Duplicate fills are idempotent. Partial fills reduce reservations only by proven executed/cancelled quantities; the remainder stays exposed. Realized PnL must be derived from the declared inventory-cost policy and exact execution/fee events, not submitted order terms. Tests must include multiple fills per order, partial-fill-then-cancel, fee in NXS, fee in USDT, unequal asset decimals, rounding/minimum boundaries, duplicate evidence and history truncation. Zero checked entities or incomplete history is unhealthy, never balanced.

## Current limitations and release evidence

- Missing open orders are marked fully filled and their full submitted volume is booked (`bot/index.js:94-125`); partial fills and cancellations can corrupt PnL.
- Open-order fetch failure returns to the tick, which can then continue toward rebalance; malformed non-array responses can become an empty authoritative-looking set.
- Balance refresh failure is logged and absorbed (`bot/index.js:75-90`); placement can use cached values.
- Order-book failure is logged and `last` price is used as the mid (`bot/index.js:51-71`), which can generate stale/crossing quotes.
- Cancellation results are discarded and all requested orders are marked cancelled (`bot/index.js:161-166`, `295-310`).
- Placement accepts a missing identity as the string `undefined`; timeouts have no durable/held outcome, do not reserve unknown exposure and do not stop later batch placements (`bot/index.js:208-230`).
- `getOpenOrders()` supplies no pair/page/cursor or completeness evidence (`bot/dextrade.js:156-165`); `getOrderHistory()` is unused.
- Millisecond `request_id` values are neither durable nor serialized and can collide across concurrent private callers (`bot/dextrade.js:114-176`).
- Price/volume precision and the 5 USDT minimum are hard-coded; admission checks and local reservations use raw volume before transmitted four-decimal rounding. A fresh boundary case admitted raw notional `5.00012` against `5.00015` available while the rounded wire notional was `5.0002`.
- PnL is aggregate weighted-average submitted value and ignores fees.
- Shared rate-limit timestamps are not serialized across concurrent callers (`bot/dextrade.js:11-22`).
- Start/stop are not serialized: `start()` awaits its first tick before installing the timer (`bot/index.js:275-292`), while `stop()` independently snapshots known IDs and reports stopped (`bot/index.js:295-317`). Fresh 2026-10-02 barrier execution again left one accepted order open after stop and installed a post-stop interval.
- Idle prefetch is admitted by one pre-await `running` check (`bot/index.js:34-47`) and then mutates the same market/balance state outside trading lifecycle serialization; startup can overlap it with the first tick.
- No automated test script or CI workflow exists. Build and syntax checks do not establish trading correctness.
- The 2026-09-21 online production audits failed with two vulnerable root package entries (`1` high, `1` low) and six bot package entries (`3` high, `2` moderate, `1` low). Fresh 2026-10-02 `npm audit --offline --omit=dev` runs reported zero in both packages, but cache-only output is not current registry evidence and does not close the prior findings. Remediation still needs compatibility and behavior gates rather than an untested lockfile rewrite.

Before unattended or meaningful-capital use, all P0 gates in [`DEVELOPMENT_PLAN.md`](DEVELOPMENT_PLAN.md) must pass, followed by P1 money/concurrency review and target dex-trade sandbox/test-account evidence. Local mocks establish containment logic only; they do not establish exchange pagination, finality, fee, cancellation or timeout-after-acceptance semantics.

See [`DEVELOPMENT_REVIEW_2026-10-02.md`](DEVELOPMENT_REVIEW_2026-10-02.md) for the current findings and executed evidence, [`DEVELOPMENT_PLAN.md`](DEVELOPMENT_PLAN.md) for the repair order, and [`ARCHITECTURE_ADDENDUM_2026-09-12.md`](ARCHITECTURE_ADDENDUM_2026-09-12.md) for the original durability decision record.
