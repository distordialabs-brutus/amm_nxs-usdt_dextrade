# Development and Architecture Review — 2026-09-17

## Baseline, delta and verdict

**Reviewed HEAD:** `7017929848d4ddef4158d868e2cd4433db09416f` on `main`.

The commit after the 2026-09-16 reviewed source HEAD (`01038ec26f8e19bb11e4eb11a7b8c1e1a2708257`) is that review's documentation publication. A path-limited diff confirms no change under `bot/`, `src/`, either package manifest or either lockfile. Runtime trees remain `bot/` `cfe5e308bbc01fc5b55329bc4378ac449720a70d` and `src/` `eac7280ff4119b667d85fa2be66d3062fc35de58`.

**Verdict: unchanged — not ready for unattended trading or meaningful capital.** No new runtime regression was introduced, but no release blocker exited. This review used only offline build, static and loopback-fake checks. It made no exchange request and used no credential, order, cancellation or trade.

## Fresh offline verification

| Gate | Result |
|---|---|
| Identity and source delta | **PASS** — HEAD `70179298…`; no runtime/manifests/lockfile delta from the 2026-09-16 reviewed source |
| Git integrity / whitespace / scope | **PASS** — `git fsck --no-dangling --no-progress`, `git diff --check`; clean baseline and only the three intended review documents changed afterward |
| Production build | **PASS** — Webpack `5.99.9`, 45.7 KiB bundle, compiled in 735 ms; stale Browserslist warning |
| Bot JavaScript syntax | **PASS** — `node --check` over all 9 tracked bot/strategy files |
| Direct production dependency trees | **PASS** — root: `nexus-module@1.1.11`, `react-redux@8.1.2`, `redux@4.1.2`; bot: `axios@1.13.5`, `cors@2.8.6`, `dotenv@16.6.1`, `express@4.22.1` |
| Root project Markdown links | **PASS** — 20 root Markdown files after this review was written; 0 missing local targets |
| Root and bot test commands | **ABSENT / FAIL** — both `npm test` invocations return `Missing script: "test"` |
| Loopback control-boundary probe | **FAIL as a safety gate** — attacker-origin requests to start/config/rebalance/stop all returned 200, exposed `Access-Control-Allow-Origin: *`, and invoked the fake controller |
| Strategy input probe | **FAIL as a safety gate** — `numGrids: 1000000` and `unknownExposureKey` were forwarded unchanged |
| Offline npm advisory lookup | Returned 0 vulnerabilities in both packages, but **not release evidence**: offline cache completeness/freshness was not established and this does not supersede the 2026-09-16 online audit failures |
| Live exchange semantics | **NOT RUN** by scope |

The loopback probe instantiated the real `bot/server.js` with an in-memory controller and an ephemeral `127.0.0.1` listener. It did not load `bot/index.js` or call dex-trade.

## Findings

1. **P0 — unauthenticated mutation and unbounded configuration remain directly executable.** `bot/server.js` still installs wildcard CORS and no control capability before all four mutation routes. The fresh probe reconfirms the issue; it is not a new regression.
2. **P0 — the prior financial-state failures remain source-identical.** `bot/index.js` still treats absence from open orders as a full fill, books submitted values as fills/PnL, absorbs balance failures, accepts missing placement IDs, and converts best-effort cancellation into universal local cancellation. The 2026-09-16 isolated execution remains the latest direct fault evidence; today's unchanged tree and static inspection confirm the code paths remain present, but those deeper temporary probes were not re-run.
3. **P0 — no import-safe controller/test boundary exists.** `bot/index.js` still constructs the exchange client, listener, timers and signal handlers at module load, while neither package exposes a test command and no CI workflow is tracked.
4. **P0/P1 — durable attribution, complete enumeration, serialization and exact money remain absent.** Process-local state, timestamp request IDs, incomplete open-order enumeration, floating-point four-decimal formatting and fill-free PnL still prevent restart-safe reconciliation.

## Prioritized coding batches

### Batch A — import-safe seam and default-deny control gate (P0)

**Targets:** reduce `bot/index.js` to process composition; add `bot/controller.js` and `bot/config.js`; update `bot/server.js`, `src/App/Main.js`, root/bot `package.json`; add `test/server.test.js`, `test/import-safety.test.js` and `.github/workflows/ci.yml`.

**Acceptance:** importing controller/server modules creates zero listener, timer, signal handler, credential load or network call; default `TRADING_ENABLED=false`; missing/wrong capability, untrusted origin, missing caps, unknown fields and million-grid input return 4xx/503 with zero controller/exchange calls; one root `npm test` command is network-denying and runs in CI.

### Batch B — evidence-typed reads and attributable writes (P0)

**Targets:** `bot/dextrade.js`, `bot/controller.js`, `bot/state.js`; add `bot/domain/orderLifecycle.js`, `test/read-evidence.test.js`, `test/order-lifecycle.test.js` and `test/cancellation.test.js`.

**Acceptance:** failed/malformed/incomplete market, balance or order scans hold the transition with zero writes/PnL; empty open-order results do not infer fills; every cancellation ID gets an exact typed result; a missing placement ID or timeout records `outcome_unknown`, reserves possible exposure and prevents every later batch/replacement call.

### Batch C — restart durability, serialization and exact fills (P0/P1)

**Targets:** add `bot/journal.js` plus a chosen durable adapter, wire it through `bot/controller.js`, and replace aggregate submitted-value PnL in `bot/state.js` with exact fill projections; add crash/restart and await-barrier fixtures.

**Acceptance:** intent precedes submission; crash points before/after remote acceptance recover one attributable action; startup adopts or holds all owned/unknown exposure before placement; tick/start/stop/config/rebalance are serialized; PnL fixtures assert exact partial-fill quantities, prices and fees. Only after these pass may capped sandbox/test-account semantics be exercised.
