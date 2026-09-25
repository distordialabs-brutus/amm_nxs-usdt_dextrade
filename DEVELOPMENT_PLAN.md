# NXS/USDT AMM Development Plan

This plan turns [`ARCHITECTURE.md`](ARCHITECTURE.md) into ordered, reviewable gates. Every implementation batch must keep the frontend build and bot syntax green, add collected regression coverage, deny unexpected network access, and preserve a default-disabled trading boundary.

## Current status — independent review 2026-09-25

Reviewed source HEAD is `e2e849cafd767ccd306b4144da2dcd5f67be8460`, equal to the locally recorded `origin/main` (`0` ahead, `0` behind). Since the 2026-09-23 reviewed source (`64185101218bf3b8b47666af07d78720b25cc9f7`), only `ARCHITECTURE.md`, `DEVELOPMENT_PLAN.md` and `DEVELOPMENT_REVIEW_2026-09-23.md` changed. Runtime trees remain `bot/` `cfe5e308bbc01fc5b55329bc4378ac449720a70d` and `src/` `eac7280ff4119b667d85fa2be66d3062fc35de58`; no manifest, lockfile, test or workflow changed. `gh run list --repo distordialabs-brutus/amm_nxs-usdt_dextrade --commit e2e849cafd767ccd306b4144da2dcd5f67be8460` returned `[]`.

Fresh offline execution passed the production build into a scratch output directory, all nine tracked bot JavaScript syntax checks, both direct production dependency-tree checks and cache-only audits. Both packages still lack a test script; no tests or `.github` workflow are tracked. Production-path probes reconfirmed attacker-origin mutation, placement after every attempted balance refresh failed, absence-based fabricated fills/PnL, missing-ID collapse, false cancellation finalization, in-memory restart loss and the stop/placement race. A new exact-units probe proved that raw admission/reservation can be lower than the rounded wire amount. Import also starts an unsynchronized idle prefetch that can overlap start. Every P0 remains open. See [`DEVELOPMENT_REVIEW_2026-09-25.md`](DEVELOPMENT_REVIEW_2026-09-25.md).

The untracked `vision.md` reviewed at SHA-256 `733a95bbb8d59d0e55acebe486a3bbbca82b70f89d66bbe0a3b50c333685c5d0` adds binding direction for this plan: preserve operator custody, make no underwriting claim, enforce a loss limit as well as order/exposure limits, fail closed under uncertain evidence, and publish inspectable release evidence. Treat it as context only until separately tracked by its owner; do not stage it with an implementation batch.

## Next coder handoff — start here, do not skip ahead

1. **Land only Batch A containment and its gate.** Create `bot/config.js` and an injected `bot/controller.js`; make `bot/index.js` composition-only; make `bot/server.js` construction side-effect free; add root-collected tests and CI. Move idle prefetch through the same scheduler/admission boundary or make it a read-only projection update with generation checks. Do not add placement/retry capability in this batch.
2. **Prove the default-off boundary through the real callers.** With no `TRADING_ENABLED`, origin, capability or complete risk policy, import and every HTTP mutation must make zero private exchange calls and install no trading timer. The test must fail if production modules reach DNS/socket/axios rather than relying on cooperative mocks.
3. **Then land Batch B as a separate acceptance-reviewed delta.** Add exact origin/capability parsing, closed request schemas and one generation-aware transition queue. Include idle prefetch, start, tick, config, rebalance, stop and shutdown in the lifecycle model. Plain `stopped` is forbidden while any placement/cancellation is accepted or unknown.
4. **Do not implement durable writes until typed reads and exact units are designed together.** Freeze exchange-derived precision/minimum metadata and exact wire units before minimum, balance, cap, reservation or journal checks. The first money test must reproduce and then reject the `2 × 2.50006` raw-versus-`2.5001` wire boundary from the current review.

## Mandatory repair order

Do not reorder these batches to add trading capability before containment and its collected gate. Do not use live exchange credentials in Batches A-D.

### Batch A — import-safe seam, one gate, default deny (P0 containment)

**Change:** reduce `bot/index.js` to composition; add injected `bot/controller.js` and `bot/config.js`; make server construction import-safe; add a root `test` command, `test/network-deny.js`, `test/import-safety.test.js`, and `.github/workflows/ci.yml`. Parse an independent `TRADING_ENABLED` switch as false unless explicitly enabled. Missing control policy or any per-order, batch, inventory, unresolved-exposure or loss policy keeps mutation disabled before controller construction can reach private exchange methods.

**Exit command:** one documented root `npm test` command runs every collected test from a clean install with real DNS/socket/axios access denied, then CI runs the same command on every push.

**Exit assertions:** importing controller, server, config and domain modules creates zero listeners, timers, signal handlers, credential reads or network calls; default configuration permits read-only liveness/status only; start/config/rebalance cause zero private calls when enablement, auth/origin or caps are missing. Re-run the 2026-09-25 import probe and require `0` for listener, interval, signal and exchange-read counts. Startup may explicitly construct the process shell, but idle prefetch must not outlive or race its admitted generation.

### Batch B — authenticated control schemas and one lifecycle queue (P0 containment)

**Change:** enforce exact configured origin plus a non-bundled control capability on every mutation route; install server-owned strategy schemas and hard per-order, batch, inventory, unresolved-outcome and loss caps. Route idle prefetch, scheduled tick, start, stop, config, forced rebalance and shutdown through one controller admission/transition protocol. Stop closes admission first, joins/resolves the active transition, reconciles attributable exposure, and prevents any later interval installation.

**Exit tests:** `test/server.test.js` rejects disabled, missing/wrong/duplicate capability, attacker/missing origin under browser policy, unknown top-level or strategy field, fractional count, `numGrids=1000000`, oversized body and over-cap notional with 4xx/503 and exactly zero controller, journal, state or exchange calls. It also proves the capability is absent from the production bundle and logs. Define and separately test the authenticated non-browser operator policy rather than implicitly trusting no `Origin` header.

`test/controller-races.test.js` places deterministic barriers before and after market, balance, open-order, journal, placement, cancellation and fill-history awaits. Record the session generation, admission state, remote-write count, timer count, journal state and quantified unresolved exposure at each barrier. Stop must linearize by changing admission to `closing`, invalidate the active generation, join the transition, and return terminal only after each attributable order is positively terminal. If placement acceptance cannot be resolved, stop returns a disabled held result; it never reports plain `stopped`, installs a timer, issues a subsequent private write or releases the reservation.

### Batch C — complete typed reads before any financial write (P0)

**Change:** return typed evidence for symbol metadata, ticker/book, balances, open orders and fill/closed history: pair, fetched-at/freshness, validated schema, canonical IDs, precision/minimums, cursor/range and explicit completeness. Remove last-trade fallback for quoting and cached-balance authorization. Open-order absence never implies fill or cancellation.

**Exit tests:** `test/read-evidence.test.js` injects transport failure, malformed envelope/row, wrong pair, stale/crossed book, missing/duplicate/non-finite balance, non-monotonic/incomplete pagination and page-budget exhaustion through the actual controller. Every case records an operator-visible hold and causes zero cancellation, placement or PnL mutation. The captured empty-open-list case must leave both orders unresolved and PnL exactly unchanged. Balance failure before the tick and the second refresh inside `rebalance()` must each permit zero placements despite a populated cached snapshot; assert both read call count and zero private writes. A valid earlier snapshot cannot satisfy freshness for a later transition.

### Batch D — attributable writes, exact cancellation cardinality, durable ambiguity (P0)

**Change:** implement the durable-intent journal path unless capped target-account evidence first proves dex-trade's idempotent client-reference and direct-lookup contract. Freeze pair, side, integer-scaled/decimal quantized price/volume, exact wire strings, notional, strategy/session generation and unique request reference before every minimum, balance, cap, reservation and submission decision. Atomically persist the intent and reserve exposure before transport. Missing ID, malformed success, timeout, interruption or ID-persistence failure becomes `outcome_unknown`, aborts the batch and blocks replacement. Cancellation has its own persisted request transition and returns one typed result per requested ID: `confirmed_cancelled`, `still_open` or `outcome_unknown`.

**Exit tests:** `test/order-lifecycle.test.js` uses a durable temporary journal and separate-process restart fixtures. Fault-inject before intent, after intent/before call, after remote acceptance/before parse, after response/before identity persistence, after persistence/before finalization, during recovery and on duplicate invocation. For every row assert exact remote-write count, frozen terms/reference/wire strings, journal state, canonical exchange identity if known, reserved base/quote exposure and admission state. An accepted-but-unresolved action remains non-sendable after restart; recovery either attaches positive authoritative evidence without another submission or retains `outcome_unknown`/`manual_review`. A `{}` response makes exactly one call, creates no `undefined` key, stops the rest of the batch and remains durable across restart. The raw/wire boundary fixture must prove one exact admitted amount is identical in policy, journal, reservation, adapter payload and reconciliation.

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
| P0 | Default-deny containment | Missing enable/auth/origin/order/exposure/loss configuration permits zero private exchange writes | **Not implemented** |
| P0 | Test/CI contract | One network-denying collected command covers controller, adapter, strategy and HTTP behavior on every push | **Not implemented** |
| P0 | Control authorization/input limits | Untrusted origin/credential and out-of-schema/over-cap requests are rejected with zero effects | **Not implemented** |
| P0 | Serialized lifecycle/safe stop | Await barriers cannot produce an open order/private write/interval after terminal stop; ambiguity returns a quantified held result | **Not implemented; failure executed 2026-09-25** |
| P0 | Fresh complete read evidence | Failed/malformed/stale/incomplete reads cause a visible hold and zero writes/PnL | **Not implemented** |
| P0 | Durable attributable writes | Ambiguous writes survive restart, retain exposure and prohibit later-batch/retry/replacement | **Not implemented** |
| P0 | Exact cancellation cardinality | Every requested ID has a typed result; only positive evidence finalizes it | **Not implemented** |
| P0 | Restart reconciliation | Startup adopts or holds all attributable/unknown exposure before placement | **Not implemented** |
| P0 | Exact wire-unit admission | Policy, reservation, journal and API payload use one quantized price/quantity/notional | **Not implemented; mismatch executed 2026-09-25** |
| P1 | Exact fill/fee accounting | Fill-level quantities, prices, decimals and fees produce deterministic exact results | **Not implemented** |
| P1 | Live exchange semantics | Capped sandbox/test-account cases prove pagination, finality and ambiguity recovery | **Not implemented** |
| P2 | Dependency remediation | Audits meet policy with build, tests and wallet compatibility green | **Deferred / compatibility-gated** |
| P2 | Operational runbook | Credentials, control capability, holds, stop and incident response are exercised | **Partial** |

**Release decision:** do not use meaningful capital or unattended operation until every P0 item passes, P1 money behavior receives independent review, and capped live-boundary evidence is complete.