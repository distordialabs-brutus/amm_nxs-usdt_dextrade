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

`bot/index.js` binds the API to `127.0.0.1`, which reduces network exposure but is not authorization. `bot/server.js` currently returns `Access-Control-Allow-Origin: *` and has no control token. The server therefore grants arbitrary origins access to start, stop, configuration and rebalance routes; actual webpage reachability also depends on the browser's private-network and mixed-content policies, which were not exercised. Do not rely on those browser policies as server authorization. The loopback boundary must not be widened, and mutating routes require an authenticated, origin-restricted control boundary before release.

## Components

- `src/App/Main.js`: dashboard, polling and operator commands.
- `bot/server.js`: local status/control API and basic input-shape validation.
- `bot/index.js`: lifecycle, non-overlapping tick, market/balance reads, reconciliation and rebalance execution. Importing it also starts the listener, which currently prevents isolated controller tests.
- `bot/state.js`: ephemeral process state; restart loses managed-order and PnL history.
- `bot/strategies/`: pure target-order generation behind a common strategy contract.
- `bot/dextrade.js`: all exchange transport, signing and rate limiting.

## Money and authority model

- Assets are NXS inventory and USDT quote currency. Amounts and prices are currently JavaScript floating-point numbers; orders are formatted to four decimal places.
- dex-trade is authoritative for order acceptance, cancellation, open/closed state, fills, execution quantities/prices, fees and account balances.
- `state.managedOrders` and `state.pnl` are process-local projections, not authoritative records. They are lost on restart.
- Every open or outcome-unknown order remains potential inventory exposure until exact exchange evidence resolves it. A timeout, malformed response, failed scan or bounded absence is not a terminal outcome.

## Invariants

1. At most one scheduled trading tick executes at a time (`tickInProgress`) — **implemented for ticks only**.
2. Only the dex-trade adapter communicates with the exchange — **implemented**.
3. A strategy computes targets but never performs I/O or mutates shared state — **implemented**.
4. Every placement batch uses a fresh, schema-valid balance and market snapshot and enforces exchange precision/minimums — **not implemented**; balance and order-book failures can be absorbed.
5. An order is terminal only from authoritative exchange evidence; absence from an open-order list is not proof of a full fill or cancellation — **not implemented**.
6. Realized PnL uses authoritative fill quantity, execution price and fees, not submitted-order values — **not implemented**.
7. Cancellation and placement have per-order identities and explicit `submitted`, `confirmed`, `outcome_unknown` and held transitions; ambiguous outcomes prohibit replacement risk — **not implemented**.
8. Control-plane requests cannot bypass exchange serialization or race cancellation/rebalance transitions — **not implemented**.
9. Mutating HTTP routes authenticate the intended wallet module and reject untrusted origins and out-of-schema strategy values — **not implemented**.
10. Private signing follows the exchange contract in `bot/dextrade.js`: SHA-256 over recursively sorted body values followed by the secret. It is not HMAC — **implemented and fixture-probed locally**.

## Evidence-gated order lifecycle

The required lifecycle is:

```text
validate control request and fresh read evidence
  -> persist/in-memory-register placement intent
  -> submit once with an attributable request identity
  -> record canonical exchange order identity
  -> resolve ambiguous outcomes from authoritative exchange evidence
  -> record exact fills/cancellation and fees
  -> permit replacement only after exposure is resolved
```

Cancellation must return one result per requested ID: `confirmed_cancelled`, `still_open`, or `outcome_unknown`. Rebalance and stop may not mark all requested orders cancelled from a best-effort batch result. A placement timeout or missing ID enters `outcome_unknown`, never blind resubmission. The controller, not logging in the adapter, owns the hold gate.

Read adapters must reject malformed envelopes, malformed records and incomplete pagination. Open-order transport/schema failure must abort the trading transition, not merely skip reconciliation and continue to place. Missing orders require exact closed-order/fill evidence. A failed balance refresh cannot authorize cached balances. Missing order-book evidence must not silently substitute last-trade price for a market-making placement decision.

State remains in memory under the existing no-database constraint. Startup therefore enters reconciliation hold until exchange evidence reconstructs attributable live orders and unresolved exposure, or an operator explicitly disposes it. If attribution cannot be proved, unattended restart is unsupported. Persistent storage requires a separate architecture decision.

## Control and concurrency boundary

The scheduled tick guard does not serialize HTTP stop/config/rebalance calls with a tick already in flight. In particular, stop can enumerate orders while a placement request is awaiting a response; that newly accepted order can escape the stop cancellation set. A single controller transition lock/queue must cover start, stop, configuration, forced rebalance, reconciliation, cancellation and placement. Remote calls must not be made while state can be concurrently finalized by another transition.

Frontend `min`, `max` and `step` fields are presentation hints only. The server must reject unknown parameters, values outside each strategy's schema, non-integer count fields and explicit resource ceilings before mutating state. The currently accepted finite-only values permit configurations such as `numGrids=1000000`.

## Current limitations and release evidence

- Missing open orders are marked fully filled and their full submitted volume is booked (`bot/index.js:94-125`); partial fills and cancellations can corrupt PnL.
- Open-order fetch failure returns to the tick, which can then continue toward rebalance; malformed non-array responses can become an empty authoritative-looking set.
- Balance refresh failure is logged and absorbed (`bot/index.js:75-90`); placement can use cached values.
- Order-book failure is logged and `last` price is used as the mid (`bot/index.js:51-71`), which can generate stale/crossing quotes.
- Cancellation results are discarded and all requested orders are marked cancelled (`bot/index.js:161-166`, `295-310`).
- Placement accepts a missing identity as the string `undefined`; timeouts have no durable/held outcome (`bot/index.js:208-230`).
- PnL is aggregate weighted-average submitted value and ignores fees.
- Shared rate-limit timestamps are not serialized across concurrent callers (`bot/dextrade.js:11-22`).
- No automated test script or CI workflow exists. Build and syntax checks do not establish trading correctness.
- The 2026-09-08 dependency audit reports two root production findings and six bot production findings; remediation needs compatibility and behavior gates.

Before unattended or meaningful-capital use, all P0 gates in [`DEVELOPMENT_PLAN.md`](DEVELOPMENT_PLAN.md) must pass, followed by P1 money/concurrency review and target dex-trade sandbox/test-account evidence. Local mocks establish containment logic only; they do not establish exchange pagination, finality, fee, cancellation or timeout-after-acceptance semantics.

See [`DEVELOPMENT_REVIEW_2026-09-08.md`](DEVELOPMENT_REVIEW_2026-09-08.md) for the current findings and executed evidence and [`DEVELOPMENT_PLAN.md`](DEVELOPMENT_PLAN.md) for the repair order.
