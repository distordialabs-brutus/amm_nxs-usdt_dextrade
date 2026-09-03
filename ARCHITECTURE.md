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

The frontend is an operator console, not the source of trading truth. The bot owns strategy selection, order placement, cancellation, reconciliation and the in-memory status snapshot. `bot/index.js` binds the control API to `127.0.0.1`; this loopback trust boundary must not be widened without authentication, origin policy and threat review.

## Components

- `src/App/Main.js`: dashboard, polling and operator commands.
- `bot/server.js`: local status/control API and input-shape validation.
- `bot/index.js`: lifecycle, non-overlapping tick, market/balance reads, reconciliation and rebalance execution.
- `bot/state.js`: ephemeral process state; restart loses managed-order and PnL history.
- `bot/strategies/`: pure target-order generation behind a common strategy contract.
- `bot/dextrade.js`: all exchange transport, signing and rate limiting.

## Invariants

1. At most one trading tick executes at a time (`tickInProgress`).
2. Only the dex-trade adapter communicates with the exchange.
3. A strategy computes targets but never performs I/O or mutates shared state.
4. Every placed order is bounded by the last refreshed available balance and the exchange minimum.
5. An order is terminal only from authoritative exchange evidence; absence from the open-order list alone is not sufficient evidence of a full fill.
6. Realized PnL is derived from authoritative fill quantity, execution price and fees, not submitted-order values.
7. Control-plane concurrency must not bypass exchange rate limits or overlap cancellation/rebalance transitions.
8. Private request signing follows the exchange contract implemented in `bot/dextrade.js`: SHA-256 over recursively sorted body values followed by the secret. It is not HMAC.

Invariants 5–7 are target requirements, not satisfied behavior as of the current review.

## Current limitations

- Missing open orders are marked fully filled and their full submitted volume is booked (`bot/index.js:105-125`), so partial fills and cancellations can corrupt PnL.
- PnL uses aggregate weighted-average buy cost and ignores fees.
- State is intentionally in memory; restart reconciliation is therefore essential before resuming trading, but is not implemented.
- API requests time out but have no bounded retry/backoff policy.
- Shared rate-limit timestamps are not serialized across concurrent callers (`bot/dextrade.js:15-22`).
- There is no automated test suite.

## Quality and release gates

The production build and JavaScript syntax checks are necessary but insufficient. Before unattended or meaningful-capital use, the repository needs fixture-backed exchange-adapter tests, authoritative fill reconciliation, restart adoption/hold behavior, fee-aware PnL tests, serialized rate limiting and failure-injection tests for timeouts, partial responses and cancellation races.

See [`DEVELOPMENT_REVIEW_2026-09-03.md`](DEVELOPMENT_REVIEW_2026-09-03.md) for current evidence and [`DEVELOPMENT_PLAN.md`](DEVELOPMENT_PLAN.md) for the repair order.
