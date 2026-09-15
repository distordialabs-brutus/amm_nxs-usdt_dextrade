# NXS/USDT AMM Development Plan

This plan turns [`ARCHITECTURE.md`](ARCHITECTURE.md) into ordered, reviewable gates. Every implementation step must keep the frontend build and bot syntax green and add automated regression coverage.

## Current status — independent review 2026-09-15

Source HEAD is `21c575206091cfb0d7117f1629a58f625d9909a9`, equal to fetched `origin/main`, with no commits after the 2026-09-12 documentation publication. There is no executable or dependency-manifest delta: `bot/` remains `cfe5e308bbc01fc5b55329bc4378ac449720a70d` and `src/` remains `eac7280ff4119b667d85fa2be66d3062fc35de58`. The fresh frontend build and nine bot syntax checks pass; production audits fail with two vulnerable root package entries and six bot package entries; no repository test command or CI workflow exists. Every P0 remains open. See the [current review](DEVELOPMENT_REVIEW_2026-09-15.md).

## Next repair batch — containment and executable seam

Keep this batch free of live exchange use and financial-state migration:

1. Make `bot/index.js` a composition shell and extract a controller factory whose imports have no listener, timer, signal, credential or network side effect. Inject exchange, scheduler, clock and ID dependencies.
2. Add parsed startup policy for `TRADING_ENABLED=false` by default, exact allowed origins, a non-bundled control capability, and hard order/notional/inventory ceilings. Route every mutation through one middleware that rejects missing policy before calling the controller.
3. Add one root test command using `node:test`, fake adapters and a network-deny guard. Cover unauthenticated/untrusted mutations, unknown and out-of-range strategy fields, missing caps, import safety and default-disabled start. Add CI for clean installs, that command, the production build, syntax, audit policy and whitespace checks.

**Batch exit:** from a clean/default environment, imports open no port and perform no network access; all mutation requests return 4xx/503 with zero controller or exchange calls; only explicit bounded test configuration reaches a fake controller; the one root gate passes without credentials. Do not begin cancellation or placement changes until this seam and gate are green.

## Ordered repair sequence

### 0. Keep the unsafe boundary contained

- Keep real exchange credentials removed and do not run unattended or with meaningful capital. Add a default-deny trading-enable gate independent of credential presence.
- When explicit enable, authenticated origin/control configuration or exposure caps are absent, mutation requests must fail closed before any private exchange call.
- **Exit:** a clean/default configuration can serve read-only health UI but every start/config/rebalance path produces zero exchange writes; only a capped disposable test configuration can enable writes.

### 1. Establish an isolated test and CI contract

- Extract controller construction and dependency injection from listener/process startup. Importing test subjects must not open a port, install signal handlers, load real credentials or call dex-trade.
- Add `node:test` coverage for strategies, server routes, controller transitions and adapter parsing/signing; use one network-denying command from a clean checkout.
- Add CI for clean installs in both packages, tests, frontend production build, bot syntax, dependency audit policy and `git diff --check`.
- **Exit:** `npm test` (or one documented root equivalent) exercises collected tests without network credentials; CI runs it on every push; a fixture that would reach the real adapter fails the test.

### 2. Contain the control plane and strategy inputs

- Require an explicit control credential on every mutating route and replace wildcard CORS with an exact configured origin policy compatible with the Nexus Wallet module. Fail startup or keep mutations disabled when the credential/origin policy is absent.
- Validate strategy names and params against server-owned schemas: reject unknown keys, below-min/above-max values, non-integer count fields and values above explicit order/loop ceilings. Enforce per-order, batch and account-exposure caps independently of strategy parameters.
- Add request body and concurrent-command tests; do not rely on browser input attributes.
- **Exit:** requests from an untrusted origin, with missing/wrong credentials, or containing `numGrids=1000000`, fractional count fields, unknown params or over-cap notional return 4xx and cause no controller call, state change or exchange call. The configured wallet origin with the right credential succeeds only inside explicit exposure caps.

### 3. Make read uncertainty stop trading

- Introduce typed results for ticker/order book, balances, open orders and closed/fill history, including schema validation, timestamps, stable pair filtering and complete-pagination evidence.
- Require fresh order-book and balance evidence before cancellation/placement. Do not fall back to last trade for quoting and do not reuse cached balances after refresh failure.
- Treat transport errors, malformed envelopes/records and page-budget exhaustion as a held state that remains visible through status.
- **Exit:** injected order-book, balance and open-order transport failures; malformed/empty-wrong-schema responses; inverted books; and incomplete/page-budget-exhausted scans produce zero cancellation, placement and fabricated PnL events and expose a reasoned hold.

### 4. Make writes attributable and replacement-safe

- Return one typed cancellation result for every requested order ID. Do not convert best-effort batch completion into universal cancellation.
- Resolve the durability decision: add a minimal durable placement-intent journal, or prove that dex-trade supplies a unique idempotent client reference and direct lookup sufficient for crash recovery. An in-memory intent is not sufficient.
- Freeze side, quantized price/volume, notional and request reference before submission. Require a validated canonical exchange ID after success; serialize reference generation and reject collisions.
- A timeout, malformed response or missing ID becomes `outcome_unknown`, reserves the possible exposure and aborts the remaining placement batch. Resolve it by positive attributable exchange evidence before retry, replacement or operator disposition.
- **Exit:** crash/fault injection before intent, after intent, after remote acceptance, before identity recording, during restart, on duplicate invocation and on partial cancellation proves exactly one attributable remote action, zero unauthorized later-batch/replacement orders and no terminal local state without exact evidence.

### 5. Serialize lifecycle transitions and stop safely

- Use one controller transition queue/lock for tick, start, stop, config and forced rebalance; keep the existing no-overlapping-ticks guard as defense in depth.
- Define stop semantics for a write already in flight and reconcile its outcome before reporting all exposure cancelled.
- Serialize public/private request scheduling so concurrent callers cannot violate rate limits or reuse timing assumptions.
- **Exit:** deterministic barriers at every await point prove stop/config/rebalance races cannot orphan an accepted order, place after a confirmed stop, exceed configured request spacing or report a false terminal state.

### 6. Reconcile restart and exact fills

- Startup begins held and completely enumerates attributable open, closed and partially filled orders before the first placement.
- Replace absence-based fill inference with exact execution quantity, price, fee, side and exchange identity evidence. Preserve unresolved liabilities.
- Keep noncritical dashboard state in memory. If exchange idempotency/lookup cannot recover a write accepted across a process crash, approve a narrowly scoped durable intent journal; otherwise unattended operation remains unsupported.
- **Exit:** restart fixtures for live owned, unknown/unowned, partial, cancelled and filled orders neither duplicate orders nor fabricate fills. Target sandbox/test-account evidence proves pagination and each terminal transition.

### 7. Rebuild PnL and remediate dependencies

- Consume target symbol metadata, quantize price/volume before minimum, balance and exposure checks, and derive inventory and realized/unrealized PnL from exact fill events using explicit decimal/base-unit policy and fee currency handling.
- Upgrade audited dependencies only under the complete behavior/build/Nexus Wallet compatibility gate.
- **Exit:** unequal-decimal, partial-fill, multi-fill, fee and rounding fixtures assert exact expected values; `npm audit --omit=dev` meets the documented release policy in both packages; production module installation and polling/control smoke pass in the target wallet.

## Status table

| Priority | Gate | Executable exit criterion | Status |
|---|---|---|---|
| P0 | Default-deny containment | Missing enable/auth/origin/cap configuration permits zero exchange writes | **Not implemented** |
| P0 | Test/CI contract | One network-denying command tests controller, adapter, strategy and HTTP behavior on every push | **Not implemented** |
| P0 | Control authorization and input limits | Untrusted origin/credential and out-of-schema requests are rejected with zero side effects | **Not implemented** |
| P0 | Fresh read evidence | Failed/malformed/incomplete market, balance or order enumeration causes an operator-visible hold and zero writes | **Not implemented** |
| P0 | Durable attributable cancellation/placement | Ambiguous writes survive restart, retain exposure and cannot trigger later-batch/retry/replacement writes without positive evidence | **Not implemented** |
| P0 | Serialized lifecycle and safe stop | Fault-injected command/tick races cannot orphan or duplicate orders | **Not implemented** |
| P0 | Restart reconciliation | Startup adopts or safely holds attributable exchange exposure before placement | **Not implemented** |
| P1 | Exact fill/fee PnL | Fill-level quantities, prices, decimals and fees produce deterministic exact results | **Not implemented** |
| P1 | Live exchange semantics | Sandbox/test-account cases prove pagination, partial fill, cancellation and timeout recovery | **Not implemented** |
| P2 | Dependency remediation | Audits meet policy with build, behavior and Nexus Wallet compatibility green | **Deferred / compatibility-gated** |
| P2 | Operational runbook | Credentials, control auth, held-state recovery, stop and incident response are documented and exercised | **Partial** |

**Release gate:** do not use meaningful capital or unattended operation until every P0 item passes, P1 money behavior has independent review, and live-boundary evidence has been collected with capped disposable test exposure.
