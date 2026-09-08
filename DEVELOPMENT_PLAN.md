# NXS/USDT AMM Development Plan

This plan turns [`ARCHITECTURE.md`](ARCHITECTURE.md) into ordered, reviewable gates. Every implementation step must keep the frontend build and bot syntax green and add automated regression coverage.

## Current status — independent review 2026-09-08

Reviewed HEAD is `18e593c37f451d3c0cbab75aec19726aead2519e`, equal to `origin/main` after fetch. The only commit since the prior review's base is the 2026-09-07 documentation publication; there is no application source delta after that review. Build, bot syntax and isolated mocked probes pass, but no repository test command or CI workflow exists. Newly documented current-tree blockers are unauthenticated wildcard-CORS control routes, server-side strategy validation/resource gaps, and trading continuation after failed order/balance/order-book evidence. See the [current review](DEVELOPMENT_REVIEW_2026-09-08.md).

## Ordered repair sequence

### 1. Establish an isolated test and CI contract

- Extract controller construction and dependency injection from listener/process startup. Importing test subjects must not open a port, install signal handlers, load real credentials or call dex-trade.
- Add `node:test` coverage for strategies, server routes, controller transitions and adapter parsing/signing; use one command from a clean checkout.
- Add CI for clean installs in both packages, tests, frontend production build, bot syntax, dependency audit policy and `git diff --check`.
- **Exit:** `npm test` (or one documented root equivalent) exercises collected tests without network credentials; CI runs it on every push; a fixture that would reach the real adapter fails the test.

### 2. Contain the control plane and strategy inputs

- Require an explicit control credential on every mutating route and replace wildcard CORS with an exact configured origin policy compatible with the Nexus Wallet module. Fail startup or keep mutations disabled when the credential/origin policy is absent.
- Validate strategy names and params against server-owned schemas: reject unknown keys, below-min/above-max values, non-integer count fields and values above explicit order/loop ceilings.
- Add request body and concurrent-command tests; do not rely on browser input attributes.
- **Exit:** requests from an untrusted origin, with missing/wrong credentials, or containing `numGrids=1000000`, fractional count fields or unknown params return 4xx and cause no controller call, state change or exchange call. The configured wallet origin with the right credential succeeds.

### 3. Make read uncertainty stop trading

- Introduce typed results for ticker/order book, balances, open orders and closed/fill history, including schema validation, timestamps and complete-pagination evidence.
- Require fresh order-book and balance evidence before cancellation/placement. Do not fall back to last trade for quoting and do not reuse cached balances after refresh failure.
- Treat transport errors, malformed envelopes/records and page-budget exhaustion as a held state that remains visible through status.
- **Exit:** injected order-book, balance and open-order transport failures; malformed/empty-wrong-schema responses; and incomplete pages produce zero cancellation, placement and fabricated PnL events and expose a reasoned hold.

### 4. Make writes attributable and replacement-safe

- Return one typed cancellation result for every requested order ID. Do not convert best-effort batch completion into universal cancellation.
- Create an in-memory placement intent before submission with an immutable request reference and frozen side, price and volume. Require a validated canonical exchange ID after success.
- A timeout, malformed response or missing ID becomes `outcome_unknown`; resolve it by positive attributable exchange evidence before retry, replacement or operator disposition.
- **Exit:** partial cancellation success, timeout after remote acceptance, missing placement ID and duplicate invocation each prove at most one remote placement, zero unauthorized replacement orders and no terminal local state without exact evidence.

### 5. Serialize lifecycle transitions and stop safely

- Use one controller transition queue/lock for tick, start, stop, config and forced rebalance; keep the existing no-overlapping-ticks guard as defense in depth.
- Define stop semantics for a write already in flight and reconcile its outcome before reporting all exposure cancelled.
- Serialize public/private request scheduling so concurrent callers cannot violate rate limits or reuse timing assumptions.
- **Exit:** deterministic barriers at every await point prove stop/config/rebalance races cannot orphan an accepted order, place after a confirmed stop, exceed configured request spacing or report a false terminal state.

### 6. Reconcile restart and exact fills

- Startup begins held and enumerates attributable open, closed and partially filled orders before the first placement.
- Replace absence-based fill inference with exact execution quantity, price, fee, side and exchange identity evidence. Preserve unresolved liabilities.
- Keep in-memory state unless a separate architecture decision approves durable storage; if exchange attribution is insufficient, require explicit operator disposition.
- **Exit:** restart fixtures for live owned, unknown/unowned, partial, cancelled and filled orders neither duplicate orders nor fabricate fills. Target sandbox/test-account evidence proves pagination and each terminal transition.

### 7. Rebuild PnL and remediate dependencies

- Derive inventory and realized/unrealized PnL from exact fill events using explicit decimal/base-unit policy and fee currency handling.
- Upgrade audited dependencies only under the complete behavior/build/Nexus Wallet compatibility gate.
- **Exit:** unequal-decimal, partial-fill, multi-fill, fee and rounding fixtures assert exact expected values; `npm audit --omit=dev` meets the documented release policy in both packages; production module installation and polling/control smoke pass in the target wallet.

## Status table

| Priority | Gate | Executable exit criterion | Status |
|---|---|---|---|
| P0 | Test/CI contract | One network-isolated command tests controller, adapter, strategy and HTTP behavior on every push | **Not implemented** |
| P0 | Control authorization and input limits | Untrusted origin/credential and out-of-schema requests are rejected with zero side effects | **Not implemented** |
| P0 | Fresh read evidence | Failed/malformed/incomplete market, balance or order enumeration causes an operator-visible hold and zero writes | **Not implemented** |
| P0 | Attributable cancellation/placement | Ambiguous writes retain exposure and cannot trigger retry/replacement without positive evidence | **Not implemented** |
| P0 | Serialized lifecycle and safe stop | Fault-injected command/tick races cannot orphan or duplicate orders | **Not implemented** |
| P0 | Restart reconciliation | Startup adopts or safely holds attributable exchange exposure before placement | **Not implemented** |
| P1 | Exact fill/fee PnL | Fill-level quantities, prices, decimals and fees produce deterministic exact results | **Not implemented** |
| P1 | Live exchange semantics | Sandbox/test-account cases prove pagination, partial fill, cancellation and timeout recovery | **Not implemented** |
| P2 | Dependency remediation | Audits meet policy with build, behavior and Nexus Wallet compatibility green | **Deferred / compatibility-gated** |
| P2 | Operational runbook | Credentials, control auth, held-state recovery, stop and incident response are documented and exercised | **Partial** |

**Release gate:** do not use meaningful capital or unattended operation until every P0 item passes, P1 money behavior has independent review, and live-boundary evidence has been collected with capped disposable test exposure.
