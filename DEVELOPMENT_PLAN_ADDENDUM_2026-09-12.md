# Development Plan Addendum — 2026-09-12

This addendum turns the 2026-09-12 review and [`ARCHITECTURE_ADDENDUM_2026-09-12.md`](ARCHITECTURE_ADDENDUM_2026-09-12.md) into executable exits. It does not mark any existing P0 complete.

## P0.0 Contain writes and establish one gate

- Add explicit write enablement that defaults false independently of exchange credential presence.
- Extract controller construction from listener/process startup and inject all exchange, clock, ID, journal, and scheduler dependencies.
- Add one network-denying root test command and CI workflow before changing financial behavior.

**Exit:** from a clean clone, the documented gate installs both packages, runs all collected tests, bot syntax, frontend production build, audits under an explicit policy, and whitespace checks. Importing test subjects opens no port, installs no signal handler, and cannot reach the network. Missing enable/auth/origin/cap policy makes every mutation request return 4xx/503 with zero adapter calls.

## P0.1 Authenticate and bound the control plane

- Replace wildcard CORS with exact configured origins.
- Require a short-lived, non-bundled local control capability for every mutation.
- Validate exact request keys and strategy-owned parameter schemas; enforce integer fields and hard order/notional/inventory/loss caps.

**Exit:** wrong/missing capability, untrusted origin, unknown keys, fractional count, `numGrids=1000000`, over-cap notional, and malformed body tests produce no controller call, state mutation, cancellation, or placement. A valid request succeeds only under independently configured caps.

## P0.2 Make every read fail closed

- Parse typed ticker, book, symbol, balance, open-order, and order/fill-history records.
- Implement complete stable pagination with page-budget exhaustion and malformed-row detection.
- Carry freshness, pair identity, and completeness into the controller; remove last-price and cached-balance fallback from write admission.

**Exit:** transport failures, malformed envelopes/rows, wrong pair, stale data, inverted/empty book, missing balances, unstable pages, duplicate cursors, and page-budget exhaustion all enter an operator-visible hold and cause zero cancellation, placement, PnL, or waterline advancement. Open-list absence alone never changes terminal order state.

## P0.3 Resolve durable identity and ambiguity

- Decide the durable-journal or proven exchange-idempotency path in the architecture addendum.
- Persist/freeze intent and reserve exposure before submission.
- Require canonical exchange identity; map timeout, malformed success, and missing identity to durable `outcome_unknown`.
- Return typed per-ID cancellation outcomes.

**Exit:** fault injection before intent, after intent, after remote acceptance, before response parsing, after identity persistence, before finalization, during restart, and on duplicate invocation proves exactly one attributable remote action. Unknown outcomes stop the remaining batch and prohibit replacement until positive evidence or recorded operator disposition resolves them. Partial cancellation failure never marks an unconfirmed order cancelled.

## P0.4 Serialize lifecycle and recover before placement

- Put tick/start/stop/config/rebalance/reconcile/cancel/place/shutdown behind one transition queue.
- Startup closes write admission, reconstructs intents, and completely enumerates attributable open and terminal exchange state.
- Stop closes admission before joining or reconciling any in-flight write.

**Exit:** deterministic barriers at every remote await prove that stop/config/rebalance races cannot place after stop, orphan accepted orders, release unknown exposure, violate request spacing, or report false terminal state. Restart fixtures for open, partially filled, filled, cancelled, unknown, duplicate, and unowned orders place nothing until reconciliation is complete.

## P1.0 Exact fills, fees, and accounting

- Adopt explicit decimal/integer scales from validated symbol metadata.
- Quantize before admission and persist transmitted values.
- Build PnL and inventory only from exact fill events, with fee amount and asset.

**Exit:** unequal-decimal, minimum-boundary, rounding, partial-fill, multi-fill, duplicate-fill, fee-in-base, fee-in-quote, and zero-liquidity fixtures assert exact values. Reconciliation detects deliberate duplicate/overpayment evidence and never turns skipped rows into green health.

## P1.1 Target-exchange acceptance

Use only a capped disposable test account after all local P0 exits pass.

**Exit:** recorded, redacted evidence proves actual symbol metadata, open/history pagination, client-reference uniqueness/idempotency or direct recovery, partial fills, cancellation finality, fees, timeout-after-acceptance recovery, rate limits, startup adoption, and safe stop. No production credential or meaningful capital is used.

## P2 Dependency and operations closure

- Upgrade audited dependencies under the complete behavior/build/wallet-install gate.
- Document credential rotation, cap changes, held-outcome disposition, restart, reconciliation, emergency stop, and incident evidence retention.

**Exit:** both production audit commands meet the documented policy; the target Nexus Wallet module install/poll/control smoke passes; operators execute a tabletop unknown-outcome and compromised-control-capability runbook without releasing unresolved exposure.

## Release rule

Unattended or meaningful-capital use remains prohibited until all P0 exits pass in CI, exact money behavior receives independent review, and P1.1 supplies target-exchange evidence. A green build or focused mock suite is not a release exit.
