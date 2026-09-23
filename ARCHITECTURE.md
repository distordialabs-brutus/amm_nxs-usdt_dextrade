# NXS/USDT AMM Architecture

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

### Current source snapshot — 2026-09-23

Reviewed source HEAD is `64185101218bf3b8b47666af07d78720b25cc9f7`, equal to local `origin/main` (`0` ahead, `0` behind) at the start of this review. The sole commit after the 2026-09-21 reviewed source (`8880f1e8a3a1800bcf547494bfac65d084b15ba9`) changes only `ARCHITECTURE.md`, `DEVELOPMENT_PLAN.md` and the new `DEVELOPMENT_REVIEW_2026-09-21.md`; `bot/` remains tree `cfe5e308bbc01fc5b55329bc4378ac449720a70d` and `src/` remains `eac7280ff4119b667d85fa2be66d3062fc35de58`. No runtime, manifest or lockfile repair landed. Every P0 release blocker remains open. Fresh offline production-path probes and deeper acceptance criteria are recorded in [`DEVELOPMENT_REVIEW_2026-09-23.md`](DEVELOPMENT_REVIEW_2026-09-23.md).

`bot/index.js` binds the API to `127.0.0.1`, which reduces network exposure but is not authorization. `bot/server.js` currently returns `Access-Control-Allow-Origin: *` and has no control token. The server therefore grants arbitrary origins access to start, stop, configuration and rebalance routes; actual webpage reachability also depends on the browser's private-network and mixed-content policies, which were not exercised. Do not rely on those browser policies as server authorization. The loopback boundary must not be widened, and mutating routes require an authenticated, origin-restricted control boundary before release.

## Components

- `src/App/Main.js`: dashboard, polling and operator commands.
- `bot/server.js`: local status/control API and basic input-shape validation.
- `bot/index.js`: lifecycle, non-overlapping tick, market/balance reads, reconciliation and rebalance execution. Importing it also starts the listener, which currently prevents isolated controller tests.
- `bot/state.js`: ephemeral process state; restart loses managed-order and PnL history.
- `bot/strategies/`: pure target-order generation behind a common strategy contract.
- `bot/dextrade.js`: all exchange transport, signing and rate limiting.

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

Fresh 2026-09-23 isolated evidence shows why this seam is the first architecture gate. With HTTP and exchange modules intercepted, importing the real `bot/index.js` still attempted the loopback listener, one interval, SIGINT/SIGTERM handlers and initial ticker, order-book and balance reads. An ephemeral loopback server built from the real `bot/server.js` forwarded all four attacker-origin mutation routes to a fake controller without authentication. A deterministic await barrier again proved the lifecycle race: stop observed no managed order while placement was pending, returned `stopped`, the accepted placement then became an uncancelled local `open` order, and `start()` installed a new trading interval after stop. These are production-path probes with external boundaries replaced, not a checked-in test suite.

The first implementation slice is deliberately narrow. Reduce `bot/index.js` to composition, move application transitions into `bot/controller.js`, centralize parsed fail-closed policy in `bot/config.js`, and keep route authorization/schema enforcement in `bot/server.js`. `src/App/Main.js` must send the configured non-bundled capability without becoming an authority source. `test/import-safety.test.js` and `test/server.test.js` must prove zero import side effects and zero controller calls for disabled, unauthenticated, wrong-origin, unknown-field or over-cap requests before order-lifecycle work begins.

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

The scheduled tick guard does not serialize HTTP stop/config/rebalance calls with a tick already in flight. The 2026-09-23 isolated barrier probe executed the concrete failure: `start()` was paused inside `createLimitOrder()`, `stop(true)` found no managed ID and returned `stopped`, then the accepted order was recorded `open`; after the original tick returned, `start()` installed another interval. A single controller transition queue must cover start, stop, configuration, forced rebalance, reconciliation, cancellation and placement. Stop must close admission, join or resolve an in-flight transition, reconcile attributable exposure, and only then report terminal state. The composition shell must not install a timer after admission has closed.

The queue requires explicit linearization semantics, not merely a mutex around route handlers. Every start creates a session generation. Stop first atomically changes admission from `open` to `closing`, invalidates that generation and cancels future timer admission. It then joins the active transition. After every awaited exchange call, the transition must re-check its generation before any further write. If a placement may have been accepted, stop must persist/recover its identity and positively cancel or classify it as a visible unresolved hold. Terminal `stopped` means no admitted transition can install a timer or issue another private call and every attributable order is terminal; otherwise the truthful result is a stopped/disabled **held** state with quantified unresolved exposure, not success.

HTTP authorization is a separate adapter concern. CORS headers are browser response policy, not authentication. Mutation must be default-disabled, require an exact configured origin and a high-entropy capability supplied at runtime (never bundled in `dist/app.js`), and reject a missing/duplicate/malformed credential before parsing or invoking controller work. Define an explicit non-browser operator path rather than treating absent `Origin` as trusted. Read routes must expose only the intended projection; logs and strategy/status fields should be reviewed for credential or sensitive exchange-data leakage. Rate limits and body-size limits supplement but do not replace authorization.

Frontend `min`, `max` and `step` fields are presentation hints only. The server must own allowlisted schemas, reject unknown top-level and strategy fields, values outside each strategy's schema, non-integer count fields and explicit per-order/batch/inventory/unresolved-exposure ceilings before mutating state. Validation and authorization failures must produce exactly zero controller, journal, state or exchange calls. The currently accepted finite-only values permit configurations such as `numGrids=1000000`.

## Authoritative reads, fills and accounting

Every write decision consumes one validated evidence bundle: pair identity, fetch time/age, complete page/range proof, canonical order IDs, symbol precision/minimum metadata, balances and a non-crossed book. Failure of any required member invalidates the whole bundle. Cached values remain display-only. An open-order list is positive evidence for rows it contains; absence is never fill/cancellation evidence without a complete authoritative closed-order/fill lookup.

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
- Price/volume precision and the 5 USDT minimum are hard-coded; admission checks use raw volume before transmitted four-decimal rounding.
- PnL is aggregate weighted-average submitted value and ignores fees.
- Shared rate-limit timestamps are not serialized across concurrent callers (`bot/dextrade.js:11-22`).
- Start/stop are not serialized: `start()` awaits its first tick before installing the timer (`bot/index.js:275-292`), while `stop()` independently snapshots known IDs and reports stopped (`bot/index.js:295-317`). Fresh 2026-09-23 barrier execution again left one accepted order open after stop and installed a post-stop interval.
- No automated test script or CI workflow exists. Build and syntax checks do not establish trading correctness.
- The 2026-09-21 online production audits failed with two vulnerable root package entries (`1` high, `1` low) and six bot package entries (`3` high, `2` moderate, `1` low). A 2026-09-23 `npm audit --offline --omit=dev` reported zero in both packages, but cache-only output is not fresh advisory evidence and does not close the prior findings. Remediation still needs compatibility and behavior gates rather than an untested lockfile rewrite.

Before unattended or meaningful-capital use, all P0 gates in [`DEVELOPMENT_PLAN.md`](DEVELOPMENT_PLAN.md) must pass, followed by P1 money/concurrency review and target dex-trade sandbox/test-account evidence. Local mocks establish containment logic only; they do not establish exchange pagination, finality, fee, cancellation or timeout-after-acceptance semantics.

See [`DEVELOPMENT_REVIEW_2026-09-23.md`](DEVELOPMENT_REVIEW_2026-09-23.md) for the current findings and executed evidence, [`DEVELOPMENT_PLAN.md`](DEVELOPMENT_PLAN.md) for the repair order, and [`ARCHITECTURE_ADDENDUM_2026-09-12.md`](ARCHITECTURE_ADDENDUM_2026-09-12.md) for the unresolved durable-intent versus proven exchange-idempotency decision.
