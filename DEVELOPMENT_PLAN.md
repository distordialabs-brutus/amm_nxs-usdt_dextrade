# NXS/USDT AMM Development Plan

This plan turns [`ARCHITECTURE.md`](ARCHITECTURE.md) into ordered, reviewable gates. Every implementation batch must keep the frontend build and bot syntax green, add collected regression coverage, deny unexpected network access, and preserve a default-disabled trading boundary.

## Current status — independent review 2026-09-23

Reviewed source HEAD is `64185101218bf3b8b47666af07d78720b25cc9f7`, equal to local `origin/main` (`0` ahead, `0` behind) at review start. Since the 2026-09-21 reviewed source (`8880f1e8a3a1800bcf547494bfac65d084b15ba9`), only `ARCHITECTURE.md`, `DEVELOPMENT_PLAN.md` and the new `DEVELOPMENT_REVIEW_2026-09-21.md` changed. Runtime trees remain `bot/` `cfe5e308bbc01fc5b55329bc4378ac449720a70d` and `src/` `eac7280ff4119b667d85fa2be66d3062fc35de58`; no manifest or lockfile changed.

Fresh offline execution passed the production build into a scratch output directory, all nine tracked bot JavaScript syntax checks, and both direct production dependency-tree checks. Both packages still lack a test script; no tests or `.github` workflow are tracked. Isolated production-path probes independently reconfirmed unauthenticated attacker-origin mutation, cached-balance placement after two failed balance reads, absence-based fabricated fills/PnL, missing-ID collapse and false cancellation finalization. A deterministic barrier again proved the stop race: an accepted order became locally open after stop returned, and the still-running `start()` call installed a post-stop interval. A two-process state probe changed one process's managed-order count from `0` to `1`; a fresh process loaded `0`, confirming that no durable intent or recovery state exists. Every P0 remains open. See [`DEVELOPMENT_REVIEW_2026-09-23.md`](DEVELOPMENT_REVIEW_2026-09-23.md).

## Mandatory repair order

Do not reorder these batches to add trading capability before containment and its collected gate. Do not use live exchange credentials in Batches A-D.

### Batch A — import-safe seam, one gate, default deny (P0 containment)

**Change:** reduce `bot/index.js` to composition; add injected `bot/controller.js` and `bot/config.js`; make server construction import-safe; add a root `test` command, `test/network-deny.js`, `test/import-safety.test.js`, and `.github/workflows/ci.yml`. Parse an independent `TRADING_ENABLED` switch as false unless explicitly enabled. Missing control policy or exposure policy keeps mutation disabled before controller construction can reach private exchange methods.

**Exit command:** one documented root `npm test` command runs every collected test from a clean install with real DNS/socket/axios access denied, then CI runs the same command on every push.

**Exit assertions:** importing controller, server, config and domain modules creates zero listeners, timers, signal handlers, credential reads or network calls; default configuration permits read-only liveness/status only; start/config/rebalance cause zero private calls when enablement, auth/origin or caps are missing. Re-run the 2026-09-23 import probe and require `0` for listener, interval, signal and exchange-read counts.

### Batch B — authenticated control schemas and one lifecycle queue (P0 containment)

**Change:** enforce exact configured origin plus a non-bundled control capability on every mutation route; install server-owned strategy schemas and hard per-order, batch, inventory and unresolved-outcome caps. Route scheduled tick, start, stop, config, forced rebalance and shutdown through one controller queue. Stop closes admission first, joins/resolves the active transition, reconciles attributable exposure, and prevents any later interval installation.

**Exit tests:** `test/server.test.js` rejects disabled, missing/wrong/duplicate capability, attacker/missing origin under browser policy, unknown top-level or strategy field, fractional count, `numGrids=1000000`, oversized body and over-cap notional with 4xx/503 and exactly zero controller, journal, state or exchange calls. It also proves the capability is absent from the production bundle and logs. Define and separately test the authenticated non-browser operator policy rather than implicitly trusting no `Origin` header.

`test/controller-races.test.js` places deterministic barriers before and after market, balance, open-order, journal, placement, cancellation and fill-history awaits. Record the session generation, admission state, remote-write count, timer count, journal state and quantified unresolved exposure at each barrier. Stop must linearize by changing admission to `closing`, invalidate the active generation, join the transition, and return terminal only after each attributable order is positively terminal. If placement acceptance cannot be resolved, stop returns a disabled held result; it never reports plain `stopped`, installs a timer, issues a subsequent private write or releases the reservation.

### Batch C — complete typed reads before any financial write (P0)

**Change:** return typed evidence for ticker/book, balances, open orders and fill/closed history: pair, fetched-at/freshness, validated schema, canonical IDs, cursor/range and explicit completeness. Remove last-trade fallback for quoting and cached-balance authorization. Open-order absence never implies fill or cancellation.

**Exit tests:** `test/read-evidence.test.js` injects transport failure, malformed envelope/row, wrong pair, stale/crossed book, missing/duplicate/non-finite balance, non-monotonic/incomplete pagination and page-budget exhaustion through the actual controller. Every case records an operator-visible hold and causes zero cancellation, placement or PnL mutation. The captured empty-open-list case must leave both orders unresolved and PnL exactly unchanged. Balance failure before the tick and the second refresh inside `rebalance()` must each permit zero placements despite a populated cached snapshot; assert both read call count and zero private writes. A valid earlier snapshot cannot satisfy freshness for a later transition.

### Batch D — attributable writes, exact cancellation cardinality, durable ambiguity (P0)

**Change:** implement the durable-intent journal path unless capped target-account evidence first proves dex-trade's idempotent client-reference and direct-lookup contract. Freeze pair, side, integer-scaled/decimal quantized price/volume, notional, strategy/session generation and unique request reference before submission. Atomically persist the intent and reserve exposure before transport. Missing ID, malformed success, timeout, interruption or ID-persistence failure becomes `outcome_unknown`, aborts the batch and blocks replacement. Cancellation has its own persisted request transition and returns one typed result per requested ID: `confirmed_cancelled`, `still_open` or `outcome_unknown`.

**Exit tests:** `test/order-lifecycle.test.js` uses a durable temporary journal and separate-process restart fixtures. Fault-inject before intent, after intent/before call, after remote acceptance/before parse, after response/before identity persistence, after persistence/before finalization, during recovery and on duplicate invocation. For every row assert exact remote-write count, frozen terms/reference, journal state, canonical exchange identity if known, reserved base/quote exposure and admission state. An accepted-but-unresolved action remains non-sendable after restart; recovery either attaches positive authoritative evidence without another submission or retains `outcome_unknown`/`manual_review`. A `{}` response makes exactly one call, creates no `undefined` key, stops the rest of the batch and remains durable across restart.

`test/cancellation.test.js` requires result cardinality and ID correspondence equal to the request. One success for A/B/C may finalize only A; B/C retain their orders and reservations. Timeout-after-cancel-acceptance remains unknown until direct evidence resolves it, and stop cannot downgrade that hold to terminal success.

### Batch E — startup adoption, exact fills and accounting (P0/P1)

**Change:** startup begins held and completely enumerates attributable open, closed, partial and unknown orders before first placement. Consume positive fill evidence with exact executed base/quote quantity, price, fee amount/asset and identity. Use explicit decimal or integer-scaled values, target symbol metadata and quantize before minimum/balance/cap checks.

**Exit tests:** crash/restart fixtures cover live owned, accepted-with-ID, accepted-with-lost-ID, unknown/unowned, partial, cancelled and filled orders without duplicate submission or fabricated terminal state. Startup performs zero placements until all journal rows are terminal or visibly held. Each recovery case asserts exact identity match, page completeness, remote-write count and retained exposure.

`test/accounting.test.js` feeds immutable fill IDs through the real reconciler: partial-fill-then-cancel, multiple fills at different prices, duplicate delivery, fee in NXS, fee in USDT, unequal asset decimals and rounding/minimum boundaries. Assert exact base/quote units, remaining reservation, inventory cost basis under one documented policy, realized/unrealized PnL and fee totals after every event. Submitted price/quantity must never substitute for missing fill fields. A deliberate duplicate or overpayment produces a positive reconciliation discrepancy; zero checked entities, malformed fill or incomplete history is unhealthy and leaves liabilities held.

### Batch F — target exchange semantics, dependencies and operations (P1/P2)

**Change:** only after A-E pass, use a capped disposable dex-trade sandbox/test account to establish pagination, pair filtering, symbol precision/minimums, fee fields, cancellation finality, idempotency/direct lookup and timeout-after-acceptance behavior. Upgrade dependencies only under the complete gate; document credentials, capabilities, held-state recovery, safe stop and incident response.

**Exit:** recorded capped test-account cases prove each external semantic that mocks cannot. `npm audit --omit=dev` meets the documented policy in both packages; build, collected tests and Nexus Wallet installation/polling/control smoke remain green. No production-readiness claim is allowed if the exchange lacks a proven recovery mechanism for ambiguous submissions.

## Acceptance evidence format

Each batch review must publish the exact source SHA, command, collected test names/count, and a requirement-to-test/code map. For financial transitions, include a compact matrix with: injected boundary; local state before/after; journal state; remote attempts and identities; exact base/quote reservation; admission/held state; and restart result. Assertions that accept any thrown error, helper-only tests that bypass the controller, or mocks that pre-decide completeness do not satisfy an exit.

Keep local containment, target-account external semantics and production operation as separate verdicts. No live exchange request is permitted before A-E pass and an explicit capped Batch F exercise is approved. Offline `npm audit` output is informational only; it cannot close a prior online advisory finding.

## Release-gate status

| Priority | Gate | Executable exit criterion | Status |
|---|---|---|---|
| P0 | Default-deny containment | Missing enable/auth/origin/cap configuration permits zero private exchange writes | **Not implemented** |
| P0 | Test/CI contract | One network-denying collected command covers controller, adapter, strategy and HTTP behavior on every push | **Not implemented** |
| P0 | Control authorization/input limits | Untrusted origin/credential and out-of-schema/over-cap requests are rejected with zero effects | **Not implemented** |
| P0 | Serialized lifecycle/safe stop | Await barriers cannot produce an open order/private write/interval after terminal stop; ambiguity returns a quantified held result | **Not implemented; failure executed 2026-09-23** |
| P0 | Fresh complete read evidence | Failed/malformed/stale/incomplete reads cause a visible hold and zero writes/PnL | **Not implemented** |
| P0 | Durable attributable writes | Ambiguous writes survive restart, retain exposure and prohibit later-batch/retry/replacement | **Not implemented** |
| P0 | Exact cancellation cardinality | Every requested ID has a typed result; only positive evidence finalizes it | **Not implemented** |
| P0 | Restart reconciliation | Startup adopts or holds all attributable/unknown exposure before placement | **Not implemented** |
| P1 | Exact fill/fee accounting | Fill-level quantities, prices, decimals and fees produce deterministic exact results | **Not implemented** |
| P1 | Live exchange semantics | Capped sandbox/test-account cases prove pagination, finality and ambiguity recovery | **Not implemented** |
| P2 | Dependency remediation | Audits meet policy with build, tests and wallet compatibility green | **Deferred / compatibility-gated** |
| P2 | Operational runbook | Credentials, control capability, holds, stop and incident response are exercised | **Partial** |

**Release decision:** do not use meaningful capital or unattended operation until every P0 item passes, P1 money behavior receives independent review, and capped live-boundary evidence is complete.