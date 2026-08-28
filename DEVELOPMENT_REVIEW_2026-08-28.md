# Development Review — 2026-08-28

## Outcome

**The prior no-development verdict remains: no repository development occurred during the review window, so no new architecture or application-code review is required.**

Review window: **2026-08-24 16:07:50 CEST through 2026-08-28 16:03:28 CEST**.

## Repository evidence

The remote was refreshed with `git fetch --prune origin` before the final history comparison.

- `git log --all --since='2026-08-24 16:07:50 +0200'` returned **zero commits**.
- `git reflog --all --since='2026-08-24 16:07:50 +0200'` returned **no entries**.
- `git stash list` returned **no stashes**.
- Local `main` and `origin/main` both resolve to `416e6524f5e358f84521cedc453787ca7ae67728` (`feat: Implement price rounding for orders and enhance error handling in bot status`, committed 2026-03-06 08:45:44 +0100).
- `git rev-list --left-right --count HEAD...@{upstream}` returned **`0 0`**.
- The only local branch is `main`. The non-main remote branch `origin/hermes-pr-test` remains at `3f54f2bb445259458f3596c2ba8f14e6fbd974d8`, dated 2026-08-02, before this review window.
- `gh pr list --state all` returned **`[]`**: there are no open, closed, or merged pull requests in the repository.
- The pre-existing untracked `DEVELOPMENT_REVIEW_2026-08-24.md` was preserved. Its birth and modification time are both 2026-08-24 16:05:23 CEST, before the cutoff, so it is not development in this window.
- Before this report was created, the worktree had no staged or unstaged tracked changes; its sole untracked file was the preserved 2026-08-24 review.

## Architecture and risk check

The current implementation still matches `README.md` and `CLAUDE.md`:

1. The React/Redux Nexus wallet module polls the localhost Express bot over HTTP every 4 seconds.
2. The bot keeps all state in memory, executes a guarded 15-second trading tick, and delegates target-order generation to registered strategy modules.
3. The bot communicates with dex-trade through the HMAC-authenticated, rate-limited REST client in `bot/dextrade.js`.

The highest-risk documented limitations also remain present in the unchanged code:

- **Fill/PnL correctness:** `bot/index.js:105-125` marks every managed order missing from the open-order response as completely filled and books its full volume. Partial fills and cancellations therefore can be misclassified, making displayed PnL approximate.
- **Cost-basis correctness:** `bot/index.js:122-125` computes realized PnL using aggregate weighted-average buy cost rather than fill-level FIFO accounting. `feesPaid` is present in state but is not incorporated into this calculation.
- **Transient exchange failures:** dex-trade requests have 10-second timeouts but no retry/backoff. Failures are either deferred to a later 15-second cycle or, for some fetches, logged while the tick continues with incomplete/stale data.
- **Rate-limit concurrency:** `bot/dextrade.js:15-22` checks and updates shared timestamps without atomic serialization. Concurrent private requests, such as stop/cancellation activity during a tick, can race.
- **Verification coverage:** `CLAUDE.md` explicitly states that the project has no tests, so these financial and concurrency behaviors lack automated regression coverage.

These are pre-existing risks, not evidence of development after the cutoff.

## Non-destructive quality gates

Per `CLAUDE.md`, no test suite was run.

| Gate | Exact result |
|---|---|
| Dependency restore | `npm ci --ignore-scripts --no-audit --no-fund` succeeded; 514 packages added. It emitted deprecation warnings for `inflight@1.0.6`, `rimraf@3.0.2`, `@babel/plugin-proposal-optional-chaining@7.21.0`, and `glob@7.2.3`. |
| Frontend production build | `npm run build` **passed**. webpack 5.99.9 compiled successfully in 882 ms and emitted `dist/js/app.js` (45.7 KiB). Browserslist warned that `caniuse-lite` data is 15 months old. The generated `dist/js/` output is ignored and did not change tracked application files. |
| Bot syntax | `node --check` **passed for all 9 tracked bot JavaScript files**: `bot/dextrade.js`, `bot/index.js`, `bot/logger.js`, `bot/server.js`, `bot/state.js`, and the four files under `bot/strategies/`. |
| Frontend dependency audit | `npm audit --json` exited 1 with **28 vulnerabilities**: 2 critical, 12 high, 10 moderate, 4 low. The critical findings are transitive `shell-quote` and `websocket-driver`; fixes are reported as available. |
| Bot dependency audit | `npm audit --json` exited 1 with **7 vulnerabilities**: 3 high and 4 moderate. Direct affected dependencies include `axios` (high) and `express` (moderate); fixes are reported as available. |
| Lint/type-check | No lint script, ESLint installation/configuration, or project TypeScript compiler is available. `tsconfig.json` is documented as editor-only (`allowJs`). |

The successful build and syntax checks show that the unchanged source still compiles/parses. The dependency-audit failures are current maintenance/security findings and should be addressed separately; they do not alter the no-development verdict.

## Documentation action

`README.md` and `CLAUDE.md` are the authoritative architecture documents; no separate architecture or development-plan file exists. Because no implementation or architecture change occurred in the review window, neither authoritative document was modified. This review file is the only new repository document created for 2026-08-28.
