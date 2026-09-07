# Development and Architecture Review — 2026-09-07

**Base HEAD:** `41c940fbbcafaf03c04fc19b22929630c220e73f` (`main`).

## Development delta and verdict

Since the previous review, the only intervening commit adds the 2026-09-04 review. No application implementation progress is evidenced. The parent review fetched origin successfully and verified zero ahead/behind. The initial worktree was clean.

**Not ready for unattended trading or meaningful capital.** This review adds concrete cancellation and malformed-evidence repair requirements to the existing reconciliation gate; these are existing source defects, not newly introduced regressions.

## Prioritized source findings

1. **P0 — cancellation results are discarded.** `bot/dextrade.js:184-193` catches individual cancellation errors and returns only successful results. `bot/index.js:161-166` ignores that result and marks every requested order cancelled before placing replacements. The stop path repeats this at `:301-309`. Failed or ambiguous cancellations can leave live originals while local state reports cancellation. Return per-order outcomes, hold unresolved orders, and prohibit replacement risk until authoritative readback confirms the relevant outcomes.
2. **P0 — malformed enumeration can become full-fill evidence.** `bot/index.js:98-110` maps a non-array response without `list` to an empty list, then marks missing managed orders filled. Validate the envelope, page completeness and each record; transport failure, malformed data and absence must never create fills. Use authoritative fill/order-history evidence before updating PnL.
3. **P0 — stale balances do not block trading.** `fetchBalances()` logs and absorbs failure (`bot/index.js:88-90`); rebalance continues using cached balances (`:169-172`). Make successful fresh balance evidence a prerequisite for placement rather than silently reusing prior state.
4. **P0 — placement identity and ambiguous acceptance are unhandled.** `bot/index.js:208-230` converts missing response IDs to the string `undefined`, and catches placement failures without a held outcome. A timeout is not proof that no order exists. Validate canonical exchange identity; hold on ambiguous acceptance; reconcile attributable orders before any replacement. Do not add blind write retries.
5. **Carry-forward:** restart adoption, authoritative fill/fee accounting, serialized exchange scheduling, cancellation/tick races and a repeatable test gate remain incomplete.

## Verification and limits

- Read the current controller, exchange adapter, state, server, strategies and prior review documents; findings above are source-trace evidence.
- Git inspection confirms no source change since the prior review and no initial dirty files.
- No repository test contract exists in either package; no test suite was claimed or run.
- The requested combined syntax/build/isolated-probe execution was **denied by command approval**. It was not retried or reformulated. No fresh build, syntax or dynamic probe pass is claimed; the previous review's passing build is historical only.
- No exchange calls, trading process, credentials or live financial mutations were used.
- Commit/push publication is not claimed: unattended command approval prevents completion of irreversible publication in this run.

## Coding sequence and acceptance

See updated [architecture](ARCHITECTURE.md) and [development plan](DEVELOPMENT_PLAN.md). First establish isolated controller/adapter tests, then implement typed evidence and held-state transitions. Required cases include partial cancellation success, timeout after remote acceptance, malformed open-order response, balance-read failure, missing placement ID, stop racing with an in-flight placement, and restart with unowned/unresolved orders. Each must prove no extra placement, no fabricated fill and an explicit operator-visible hold. Preserve HTTP polling, the tick guard, centralized adapter, and the existing in-memory/no-database constraint; exchange reconstruction or explicit operator disposition must precede restart trading.
