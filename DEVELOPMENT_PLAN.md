# NXS/USDT AMM Development Plan

This plan turns [`ARCHITECTURE.md`](ARCHITECTURE.md) into ordered, reviewable gates. Each step must leave the frontend build and bot syntax green and add automated regression coverage.

## Current status — independent review 2026-09-07

No source development occurred since the preceding review. Historical build/syntax passes are not fresh verification: this run's combined execution was approval-denied and not retried. No automated test contract exists, and financial correctness is not release-ready. Source inspection adds explicit cancellation-result, malformed-enumeration, stale-balance and ambiguous-placement gates; see [current review](DEVELOPMENT_REVIEW_2026-09-07.md).

### First repair batch: make uncertainty stop replacement trading

- Extract controller construction from server/listener startup so mocked adapter tests cannot reach the exchange.
- Make cancellation return a result for every requested ID. Partial failure or timeout leaves unresolved orders held; neither stop nor rebalance may fabricate cancellation.
- Validate open-order schemas and completeness before reconciliation. An empty or malformed result must not book full fills.
- Require a successful fresh balance snapshot for each placement batch.
- Validate placement response IDs. A missing ID or timeout after acceptance enters `outcome_unknown`; resolve through attributable exchange evidence, never blind resubmission.
- Acceptance: injected partial cancellation, malformed enumeration, stale balances, missing IDs, accepted-but-timed-out placement, and stop/placement races cause zero unauthorized replacement orders and zero fabricated fill/PnL entries. Assert operator-visible held state and explicit recovery behavior.
- Preserve the in-memory state rule. Restart must reconstruct from exchange evidence or remain held for operator disposition; any durable-store proposal needs a separate architecture decision.

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
