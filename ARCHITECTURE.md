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

### Current source snapshot — 2026-09-17

Repository HEAD is `7017929848d4ddef4158d868e2cd4433db09416f`. The only commit after the 2026-09-16 reviewed source HEAD (`01038ec26f8e19bb11e4eb11a7b8c1e1a2708257`) is that review's documentation publication. A path-limited diff confirms no executable, manifest or lockfile delta: `bot/` remains `cfe5e308bbc01fc5b55329bc4378ac449720a70d` and `src/` remains `eac7280ff4119b667d85fa2be66d3062fc35de58`. Every P0 release blocker therefore remains open. Fresh offline verification and the re-executed control-boundary failure are recorded in [`DEVELOPMENT_REVIEW_2026-09-17.md`](DEVELOPMENT_REVIEW_2026-09-17.md).

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

Fresh isolated evidence shows why this seam is the first architecture gate. With HTTP and exchange modules intercepted, importing the real `bot/index.js` still attempted the loopback listener, one interval, SIGINT/SIGTERM handlers and initial ticker, order-book and balance reads. An ephemeral server built from the real `bot/server.js` also forwarded all four attacker-origin mutation routes to a fake controller without authentication. These are production-path probes with external boundaries replaced, not a checked-in test suite.

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

Read adapters must reject malformed envelopes, malformed records and incomplete pagination. `getOpenOrders()` currently has no pagination arguments or completeness result, while reconciliation treats its returned set as complete. Open-order transport/schema/page-budget failure must abort the trading transition, not merely skip reconciliation and continue to place. Missing orders require exact closed-order/fill evidence. A failed balance refresh cannot authorize cached balances. Missing, inverted or stale order-book evidence must not silently substitute last-trade price for a market-making placement decision.

Ordinary dashboard state can remain in memory under the existing no-database constraint, but an in-memory placement intent cannot survive a crash after remote acceptance. Before unattended trading, the architecture must either approve a minimal durable intent journal or prove that the exchange provides a unique idempotent client reference and direct authoritative lookup sufficient to recover every ambiguous submission. Startup remains held until that evidence reconstructs attributable live orders and unresolved exposure, or an operator explicitly disposes it. Without one of those recovery mechanisms, unattended restart is unsupported.

## Control and concurrency boundary

The scheduled tick guard does not serialize HTTP stop/config/rebalance calls with a tick already in flight. In particular, stop can enumerate orders while a placement request is awaiting a response; that newly accepted order can escape the stop cancellation set. A single controller transition lock/queue must cover start, stop, configuration, forced rebalance, reconciliation, cancellation and placement. Remote calls must not be made while state can be concurrently finalized by another transition.

Frontend `min`, `max` and `step` fields are presentation hints only. The server must reject unknown parameters, values outside each strategy's schema, non-integer count fields and explicit resource ceilings before mutating state. The currently accepted finite-only values permit configurations such as `numGrids=1000000`.

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
- No automated test script or CI workflow exists. Build and syntax checks do not establish trading correctness.
- Fresh 2026-09-16 production audits fail with two vulnerable root package entries (`1` high, `1` low) and six bot package entries (`3` high, `2` moderate, `1` low); remediation needs compatibility and behavior gates.

Before unattended or meaningful-capital use, all P0 gates in [`DEVELOPMENT_PLAN.md`](DEVELOPMENT_PLAN.md) must pass, followed by P1 money/concurrency review and target dex-trade sandbox/test-account evidence. Local mocks establish containment logic only; they do not establish exchange pagination, finality, fee, cancellation or timeout-after-acceptance semantics.

See [`DEVELOPMENT_REVIEW_2026-09-17.md`](DEVELOPMENT_REVIEW_2026-09-17.md) for the current findings and executed evidence, [`DEVELOPMENT_PLAN.md`](DEVELOPMENT_PLAN.md) for the repair order, and [`ARCHITECTURE_ADDENDUM_2026-09-12.md`](ARCHITECTURE_ADDENDUM_2026-09-12.md) for the unresolved durable-intent versus proven exchange-idempotency decision.
