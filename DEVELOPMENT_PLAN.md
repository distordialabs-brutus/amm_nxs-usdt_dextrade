# NXS/USDT AMM Development Plan

This plan turns [`ARCHITECTURE.md`](ARCHITECTURE.md) into ordered, reviewable gates. Each step must leave the frontend build and bot syntax green and add automated regression coverage.

## Current status — independent review 2026-09-03

No source development occurred after 2026-09-02 16:22:20 +0200. Build and syntax pass, but no automated test command exists and financial correctness is not release-ready.

| Priority | Step | Exit criterion | Status |
|---|---|---|---|
| P0 | Establish test/CI contract | One command tests bot state transitions, exchange adapter, strategies and control API on every push | **Not implemented** |
| P0 | Authoritative order reconciliation | Closed-order/fill evidence distinguishes full fill, partial fill and cancellation; ambiguous evidence holds without booking | **Not implemented** |
| P0 | Restart reconciliation | Startup adopts or safely holds live bot-owned orders before any new placement | **Not implemented** |
| P1 | Fee-aware PnL ledger | Fill-level quantities, prices and fees produce deterministic realized/unrealized PnL | **Not implemented** |
| P1 | Serialized exchange scheduling | Concurrent private/public calls cannot violate rate limits; cancellation and rebalance races are tested | **Not implemented** |
| P1 | Failure policy | Bounded retry/backoff and idempotency rules are explicit per read/write endpoint | **Not implemented** |
| P2 | Dependency remediation | Frontend/bot audit findings are repaired without breaking Nexus module compatibility | **Deferred / compatibility-gated** |
| P2 | Documentation accuracy | SHA-256 signing terminology is consistent; operational runbook covers credentials, stop and recovery | **Partial** |

## Required sequence

1. Add a deterministic Node test runner and CI gate without changing trading behavior.
2. Capture representative dex-trade fixtures for open orders, order history, partial fills, cancellations, errors and pagination.
3. Replace absence-based fill inference with authoritative reconciliation and an explicit ambiguous/held state.
4. Add startup order adoption/hold before the first trading tick.
5. Rebuild PnL from fill-level quantity, price and fee events.
6. Serialize request scheduling and test stop/rebalance concurrency.
7. Add endpoint-specific retry/idempotency behavior and failure injection.
8. Remediate dependencies only with build, Nexus Wallet integration and behavior tests green.

**Release gate:** do not use meaningful capital or unattended operation until P0 items pass and P1 money/concurrency behavior has independent review evidence.
