# Development and Architecture Review — 2026-09-23

## Baseline, scope and verdict

**Reviewed source HEAD:** `64185101218bf3b8b47666af07d78720b25cc9f7` on `main`, equal to local `origin/main` (`0` ahead, `0` behind) at review start.

The sole commit after the 2026-09-21 reviewed source (`8880f1e8a3a1800bcf547494bfac65d084b15ba9`) changes only `ARCHITECTURE.md`, `DEVELOPMENT_PLAN.md` and the new `DEVELOPMENT_REVIEW_2026-09-21.md`. Runtime trees remain `bot/` `cfe5e308bbc01fc5b55329bc4378ac449720a70d` and `src/` `eac7280ff4119b667d85fa2be66d3062fc35de58`. No executable, manifest or lockfile repair landed.

**Verdict: unchanged—unsafe for unattended trading or meaningful capital.** This documentation review does not claim new implementation. It made no dex-trade request, loaded no real credential and created/cancelled no exchange order. Production paths were exercised only with synthetic credentials, intercepted exchange/process adapters, an ephemeral loopback HTTP listener and scratch output under the active Hermes profile.

## Executed offline evidence

The temporary probe required the real `bot/server.js`, `bot/index.js`, `bot/state.js` and strategies. For controller cases, it replaced `bot/dextrade.js` and process HTTP/timer boundaries before module load. The script is review evidence in the profile scratch directory, not a tracked or collected test.

| Gate / command | Result |
|---|---|
| `git diff --name-status 8880f1e8a3a1800bcf547494bfac65d084b15ba9..HEAD` | Only the three 2026-09-21 review documents; runtime trees unchanged |
| `npm run build -- --output-path /home/brutus/.hermes/profiles/principal-dev/cache/scratch/amm-build-2026-09-23-run2` | **PASS**—Webpack `5.99.9`, 45.7 KiB app bundle, compiled successfully in 775 ms; stale Browserslist-data warning |
| `node --check` for tracked `bot/*.js` and `bot/strategies/*.js` | **PASS**—`syntax_checked=9` |
| root `npm test` | **FAIL / absent**—exit `1`, `Missing script: "test"` |
| `cd bot && npm test` | **FAIL / absent**—exit `1`, `Missing script: "test"` |
| `git ls-files '.github/**' '*test*' '*spec*'` | No tracked CI workflow or test/spec candidate |
| root `npm ls --depth=0 --omit=dev` | **PASS**—`nexus-module@1.1.11`, `react-redux@8.1.2`, `redux@4.1.2` |
| bot `npm ls --depth=0 --omit=dev` | **PASS**—`axios@1.13.5`, `cors@2.8.6`, `dotenv@16.6.1`, `express@4.22.1` |
| root and bot `npm audit --offline --omit=dev` | Returned `found 0 vulnerabilities`; cache-only audit is not fresh advisory evidence and does not close the 2026-09-21 online failures |
| Loopback control probe | Four attacker-origin mutation requests (`start`, `config`, `rebalance`, `stop`) each returned 200, `Access-Control-Allow-Origin: *`, and invoked the fake controller; `numGrids: 1000000` plus `unknownExposureKey: 7` passed through |
| Import probe | Requiring `bot/index.js` attempted 1 listener, 1 interval, 2 signal registrations and initial ticker/book/balance reads (`1` each) |
| Failed-balance probe | Both balance reads failed; 2 orders were still placed from preloaded cached balances |
| Negative-open-list probe | Empty open list changed both seeded orders to `filled` and booked buy volume `10`, sell volume `10`, buy cost `10`, sell revenue `12`, realized PnL `2`, fees `0` |
| Missing-ID probe | Two `{}` create responses produced one key named `undefined`; the second order overwrote the first |
| Cancellation-cardinality probe | Three requested IDs and one returned success caused all A/B/C local rows to become `cancelled` |
| Stop barrier probe | At stop return: status `stopped`, no known orders, 0 cancellation calls, 0 timer installs. After the pending acceptance resolved: `accepted-after-stop` was `open`, cancellation calls remained 0, and 1 interval had been installed |
| Two-process state probe | First process seeded 1 managed order; a fresh process loaded 0, confirming that managed orders/intents do not survive restart |
| Live exchange semantics | **NOT RUN** by scope |

## Findings in financial-risk order

### 1. P0—terminal stop is not serialized with in-flight placement

`start()` marks the bot running, awaits its first `tick()`, then unconditionally installs the interval (`bot/index.js:275-292`). `stop()` independently clears the current timer, snapshots only already-recorded IDs and reports stopped (`bot/index.js:295-317`). Placement does not record an ID until `createLimitOrder()` resolves (`bot/index.js:208-230`).

The fresh barrier execution stopped while that await was pending. Stop returned terminal with nothing to cancel; the accepted order was then recorded open, and the old start transition installed a new timer. A `tickInProgress` boolean only prevents overlapping scheduled ticks (`bot/index.js:238-270`); it does not serialize lifecycle commands or provide a stop linearization point.

**Required acceptance:** one queue must admit tick/start/stop/config/rebalance/shutdown transitions. Stop first closes admission and invalidates the session generation, then joins the active transition. Barriers before and after every journal/exchange await must prove that plain `stopped` implies zero attributable open/unknown exposure, zero subsequent private calls and zero later timer installation. An accepted-but-unresolved placement must produce a disabled held result with quantified exposure, not terminal success.

### 2. P0—mutation is unauthenticated and server-side schemas/caps are absent

Wildcard CORS is installed at `bot/server.js:31-36`; all mutation routes at `bot/server.js:56-105` lack a capability check. Validation accepts every finite numeric strategy field (`bot/server.js:16-29`) and ignores the strategy metadata's min/max/step contract. The frontend sends only JSON content type (`src/App/Main.js:64-68`) and has no runtime control capability.

Loopback binding reduces reachability but does not authenticate browser or local-process callers. CORS is response policy, not authority. Browser private-network/mixed-content behavior was not tested and is not a valid server-side safety control.

**Required acceptance:** default-disabled mutation, exact configured browser origin, non-bundled high-entropy capability, an explicit authenticated no-Origin operator policy, server-owned allowlisted schemas and hard per-order/batch/inventory/unresolved-exposure caps. Disabled, missing/wrong/duplicate credential, attacker/missing origin, unknown field, fractional count, million-grid, oversized body and over-cap tests must each assert 4xx/503 and exactly zero controller, journal, state and exchange calls. The production bundle and logs must not contain the capability.

### 3. P0—failed balance reads still authorize orders from cache

`fetchBalances()` catches and absorbs every error (`bot/index.js:75-90`). A tick calls it once before rebalance (`bot/index.js:246-258`), and rebalance calls it again before reading `state.balances` (`bot/index.js:169-172`). Neither call returns typed freshness/success evidence. The fresh probe forced both reads to fail and still observed two placements funded by preloaded cached balances.

**Required acceptance:** a financial transition consumes one pair-bound, schema-valid, fresh balance evidence object. Either failed read—before strategy decision or after cancellation—must atomically hold the transition, preserve current reservations, emit operator-visible health and permit zero placement/cancellation/PnL side effects. Tests must preload apparently sufficient cached balances so they cannot pass accidentally with zero funds.

### 4. P0/P1—open-list absence fabricates fills and PnL from submitted terms

`reconcileOrders()` interprets absence from `getOpenOrders()` as a complete fill and books submitted volume and price (`bot/index.js:94-125`). `getOpenOrders()` provides no pair, cursor/page or completeness evidence (`bot/dextrade.js:156-165`). Although `getOrderHistory()` exists (`bot/dextrade.js:167-179`), production has no caller. The probe's empty list turned both seeded orders into fills and invented 2 USDT realized PnL without a fill ID, execution quantity/price or fee.

**Required acceptance:** absence from open orders remains unresolved. Terminal/accounting transitions require complete positive closed/fill evidence keyed by immutable fill ID, with exact executed base/quote quantity, fee amount/asset and authoritative timestamp. Collected fixtures must cover multi-fill, partial-fill-then-cancel, duplicate delivery, fees in NXS and USDT, unequal decimals, rounding/minimum boundaries, malformed/truncated history and zero checked entities. Assert exact remaining reservations and cost-basis/PnL after every event; never substitute submitted terms.

### 5. P0—writes have no durable identity or restart recovery protocol

Private request IDs are `String(Date.now())` generated immediately before each call (`bot/dextrade.js:114-176`). Placement turns a missing exchange ID into the literal key `undefined`, catches errors and continues the batch (`bot/index.js:208-230`). `bot/state.js:6-49` stores all lifecycle and PnL data in one process-local object. The two-process probe confirmed total loss, and search found no production `intent` or `outcome_unknown` state.

This means a timeout or crash after remote acceptance cannot be distinguished from rejection. Startup does not enumerate/adopt attributable open, closed or partial orders before trading. Retrying risks duplicate exposure; continuing without retry can strand an accepted order.

**Required acceptance:** atomically persist frozen quantized terms, a durable unique client reference and exact reservation before transport. Treat timeout, malformed success, interruption and identity-persistence failure as `outcome_unknown`; stop the batch and prohibit replacement. Separate-process fault fixtures must cover every pre-intent/submit/accept/parse/identity/finalize/restart boundary and assert exact remote attempt count, journal state, retained base/quote exposure and admission state. Recovery may attach only positive direct identity or complete attributable history evidence; otherwise retain a non-sendable manual hold.

### 6. P0—cancellation result cardinality and finality are discarded

`DexTradeClient.cancelAll()` omits failed IDs from its return (`bot/dextrade.js:181-193`). Rebalance and stop ignore returned evidence and mark every requested local row cancelled (`bot/index.js:161-166,301-310`). The fresh three-ID probe returned one success and nevertheless finalized all three.

**Required acceptance:** cancellation returns exactly one ID-correlated typed result per request: `confirmed_cancelled`, `still_open` or `outcome_unknown`. Only positive authoritative evidence releases exposure. Timeout-after-acceptance remains unknown through stop/restart until resolved. One success for A/B/C must finalize only A and preserve B/C rows and reservations.

### 7. P1—there is still no engineering gate or external-semantics evidence

The production build and syntax pass, but neither package defines tests and no CI/test files are tracked. Importing the composition module performs process and network work (`bot/index.js:334-372`), making isolated production-path tests harder. The offline audit result is cache-dependent and cannot supersede the prior online advisory findings. No target-account evidence establishes dex-trade pagination, pair filtering, precision/minimums, fee schema, cancellation finality, client-reference idempotency/direct lookup or timeout-after-acceptance behavior.

## Positive controls actually verified

- The HTTP listener remains loopback-bound in `bot/index.js:338-339`.
- Exchange transport remains centralized in `bot/dextrade.js`; strategies remain I/O-free order generators.
- Scheduled ticks retain a non-overlap guard, though it is not a lifecycle lock.
- The frontend production build and all nine tracked bot JavaScript syntax checks pass.
- The review started from a clean worktree; runtime source exactly matches the prior reviewed runtime trees.

## Repair order

1. **Contain and create one collected gate:** import-safe composition, injected controller/config/exchange/journal/clock/scheduler, default-false trading switch, network denial and CI.
2. **Authenticate and serialize:** strict mutation authentication/schemas/caps plus one generation-aware transition queue and truthful held stop result.
3. **Require complete fresh reads:** typed pair/freshness/schema/pagination evidence; cached display state cannot authorize writes; negative open-list evidence cannot finalize.
4. **Persist attributable intent:** intent-first reservations, unique references, held ambiguity, exact cancellation cardinality and separate-process fault recovery.
5. **Adopt and account from real fills:** startup hold, complete attribution, exact decimals, fill/fee identities, deterministic reservations and PnL.
6. **Only then exercise capped target semantics and dependencies:** approved disposable test account, timeout-after-acceptance/finality/pagination fixtures, online audit remediation and Nexus Wallet compatibility.

Detailed executable exits and the required evidence matrix are maintained in [`DEVELOPMENT_PLAN.md`](DEVELOPMENT_PLAN.md). This review was not committed or pushed.