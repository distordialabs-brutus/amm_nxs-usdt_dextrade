# NXS/USDT Market-Making Infrastructure — Vision

## Portfolio roadmap authority

The master Distordia project also owns `PORTFOLIO_DEVELOPMENT_PLAN.md` and its strategy-decision register. The full order is **master strategy/customer evidence → portfolio roadmap/decisions → this vision → architecture/development plan → tasks/code/tests/release evidence**. Read the [portable repository alignment](docs/DISTORDIA_ALIGNMENT.md) for objective, customer-evidence, ownership and dependency mapping. Master-source paths below are local workspace references, not promised GitHub links. This section adds portfolio sequencing; it does not certify the envisioned behavior or amend unresolved master strategy assumptions.

## Accountability venture context — not canonical authority

[Staked Accountability Rails](../../projects/Distordia/staked-accountability-rails.md) and [Infrastructure Buildout](../../projects/Distordia/infrastructure-buildout.md) are venture hypotheses and dependency-design context. They do not amend canonical strategy or prove enforceable collateral/slashing, non-custody, regulatory status, reputation, or adoption. Interpret unresolved claims through the master portfolio decision register (SD-002–SD-008); feasibility, legal assessment and human decisions remain required.

## Purpose

This repository should provide operator-controlled market-making infrastructure for the NXS/USDT market. Its role is to make exchange access, price discovery, and settlement more usable for the Nexus and Distordia ecosystems through bounded, observable liquidity operations.

Liquidity is an enabling capability, not Distordia's strategic center. The durable value is trustworthy coordination: explicit authority, accountable actions, reconcilable settlement, and evidence that operators and counterparties can independently inspect.

Today the repository is a Nexus Wallet console and bot that submits orders to dex-trade.com. It is not an on-chain AMM, a decentralized exchange, or production-ready financial infrastructure. The vision must be advanced by closing that gap through evidence, not by changing the label.

## Operating doctrine

- **Operator-controlled capital.** Operators retain custody of their exchange accounts, choose strategies and limits, enable writes explicitly, and can stop or hold the system. The software must not pool customer funds or become a discretionary custodian.
- **No underwriting.** Distordia does not guarantee liquidity, execution, price, uptime, principal, or returns and does not absorb trading losses. Market and counterparty risk remain explicit with the capital owner.
- **Bounded authority.** Strategy code proposes orders only within independently enforced per-order, batch, inventory, loss, and unresolved-exposure limits. Missing policy or evidence means no financial write.
- **Fail closed under uncertainty.** Stale, malformed, partial, or incomplete exchange data cannot authorize action. Ambiguous submissions, cancellations, fills, or restarts enter a visible held state until resolved by positive evidence or recorded human disposition.
- **Deterministic money behavior.** Prices, quantities, fees, reservations, and PnL use explicit decimal or integer-scaled arithmetic and exchange-provided precision. Results must be reproducible from immutable inputs.
- **Reconciliation before activity.** The exchange is authoritative for acceptance, fills, fees, balances, and cancellation. Durable intent precedes submission; startup and recovery reconcile every attributable or outcome-unknown order before new placement.
- **Human authority at consequential gates.** Operators control write enablement, capital limits, unresolved-outcome disposition, credential changes, and release approval. Automation may not silently widen its own mandate.
- **Transparent evidence.** Decisions, intent identities, exchange evidence, held reasons, exposure, and release-gate results should be inspectable without exposing credentials. Assertions of safety or readiness require reproducible tests and capped target-environment evidence.
- **Open edges.** Interfaces, records, and evidence formats should be documented and portable so ecosystem participants can verify behavior without depending on a proprietary dashboard.

## Desired ecosystem outcome

The project succeeds when an operator can provide modest, deliberately bounded liquidity while preserving truthful state through failures, races, partial fills, restarts, and uncertain exchange responses. Market-quality indicators may guide operations, but volume and strategy profit never override safety limits or evidentiary integrity.

This capability can support ecosystem access and settlement while Distordia concentrates on the scarcer layers of trust, verification, accountability, and coordination. It is infrastructure for those goals—not a promise of investment performance and not a reason for Distordia to take custody or underwrite risk.

## Authority hierarchy

When documents or implementation disagree, use this order:

1. **Canonical master strategy and customer evidence** — [Business Thesis and Strategy v2](../../projects/Distordia/Distordia_Labs_Business_Thesis_and_Strategy_v2.docx) and [Customer Problem Atlas v2](../../projects/Distordia/Distordia_Customer_Problem_Atlas_v2.docx). The master `PORTFOLIO_DEVELOPMENT_PLAN.md` records portfolio sequencing and explicit strategy decisions before this repository vision.
2. **This repository vision** — purpose, boundaries, and durable operating doctrine.
3. **Architecture and development plan** — [`ARCHITECTURE.md`](ARCHITECTURE.md), [`ARCHITECTURE_ADDENDUM_2026-09-12.md`](ARCHITECTURE_ADDENDUM_2026-09-12.md), [`DEVELOPMENT_PLAN.md`](DEVELOPMENT_PLAN.md), and its addendum translate the vision into design and ordered gates.
4. **Code, tests, and deployment evidence** — establish only what is actually implemented and verified. They cannot silently redefine higher-level intent.

Higher-level documents govern direction; lower-level evidence governs claims about current behavior. A conflict is resolved explicitly in the appropriate higher-level document rather than rationalized in code.

## Development grounding checklist

For every material change or release decision:

- [ ] Name the ecosystem purpose served; reject work that treats trading volume, complexity, or speculation as the goal.
- [ ] Preserve operator custody and human authority; introduce no pooled capital, hidden delegation, risk guarantee, or Distordia balance-sheet exposure.
- [ ] Keep financial writes default-disabled and require authenticated control plus explicit, independently enforced risk limits.
- [ ] Define affected invariants, failure modes, maximum exposure, held behavior, and operator recovery before implementation.
- [ ] Use fresh, typed, pair-bound, schema-valid, complete exchange evidence; absence is not proof of a terminal outcome.
- [ ] Persist attributable intent before transport and retain `outcome_unknown` exposure across timeout, crash, and restart without blind retry.
- [ ] Serialize lifecycle transitions so stop, config, rebalance, reconciliation, cancellation, and placement cannot race authority.
- [ ] Quantize before limit and balance checks; derive fills, fees, inventory, and PnL only from authoritative evidence using exact arithmetic.
- [ ] Expose reconciled state, held reasons, unresolved exposure, and evidence provenance while redacting credentials and capabilities.
- [ ] Add adversarial tests for malformed data, incomplete pagination, partial fills, duplicate evidence, ambiguous writes, cancellation cardinality, races, and restart recovery.
- [ ] Separate local containment proof from capped target-exchange validation; use no meaningful capital before all release gates pass.
- [ ] Record the exact source revision, commands, results, and unresolved risks. A build, mock, backtest, or profitable run alone is not a readiness claim.
- [ ] Check the authority hierarchy and update architecture or plans when assumptions change; never let implementation drift become strategy.
