# Architecture Addendum — 2026-09-12

This addendum supplements [`ARCHITECTURE.md`](ARCHITECTURE.md). It does not describe implemented behavior: source is unchanged since the 2026-09-10 review baseline.

## Release architecture decision

Unattended trading requires an attributable, restart-safe financial state machine. The current in-memory-only constraint cannot satisfy that requirement unless dex-trade supplies a proven unique idempotent client reference plus direct authoritative lookup for every submission outcome.

Choose and document exactly one path before placement work proceeds:

1. **Durable intent path:** approve a minimal transactional journal for immutable placement/cancellation intents, exchange identities, unknown outcomes, exposure reservations, and operator dispositions; or
2. **Exchange-idempotency path:** prove in the target environment that a unique client reference is enforced idempotently, survives timeout/restart, and supports direct lookup without relying on bounded history absence.

If neither path is proven, restartable or unattended trading remains unsupported. Ordinary dashboard snapshots may remain in memory; unresolved financial intent may not.

## Target trust and process boundary

```text
Nexus Wallet module
  | exact allowed origin + non-bundled control capability
  | status reads; authenticated mutation commands
  v
Loopback control API
  | server-owned request schemas and exposure policy
  v
Single serialized controller transition queue
  | durable intent / outcome-unknown reservation
  v
Dex-trade adapter
  | validated complete reads; attributable writes; typed outcomes
  v
Authoritative dex-trade order, fill, fee, and balance state
```

The control capability must not be a static secret shipped in the frontend bundle. Prefer a bot-minted, short-lived capability delivered through a trusted local launch/IPC boundary. Exact-origin checks are additional policy, not authentication. Missing enablement, authentication, origin, cap, or strategy policy keeps every mutation route fail-closed.

## Authoritative evidence model

Every exchange read returns a typed evidence object containing:

- endpoint and pair identity;
- fetch timestamp and freshness decision;
- schema-validation result;
- pagination/cursor range and explicit completeness;
- canonical exchange IDs;
- source error category without credential leakage.

A transport error, malformed envelope or row, page-budget exhaustion, unstable scan, wrong pair, stale snapshot, crossed/inverted book, or missing required balance is `held`, never an empty success. Only the component that proves complete enumeration may advance a scan waterline.

Open-order absence cannot establish `filled` or `cancelled`. A terminal transition needs positive order/fill history keyed to the exact exchange ID and must preserve exact executed base/quote quantities, execution prices, fee amount/asset, and status.

## Write protocol

```text
validate enablement, auth, schema, caps, and fresh complete reads
  -> freeze pair, side, quantized price/volume, notional, and client reference
  -> persist immutable intent and reserve exposure
  -> submit once
  -> persist canonical exchange identity or outcome_unknown
  -> reconcile by positive attributable exchange evidence
  -> persist exact fills/fees/cancellation
  -> release exposure and permit replacement
```

A timeout, process interruption, malformed success, missing order ID, or transport failure after submission is `outcome_unknown`. It aborts the remaining placement batch and prohibits retry, replacement, refund-like compensation, or exposure release until authoritative positive evidence or explicit operator disposition resolves it.

Cancellation returns one result per requested ID: `confirmed_cancelled`, `still_open`, or `outcome_unknown`. A best-effort batch return may not be converted into universal success.

## Concurrency and lifecycle

One controller-owned queue/lock covers start, stop, config, scheduled tick, forced rebalance, reconciliation, cancellation, placement, and shutdown. Stop first closes admission, then joins or resolves any in-flight write, enumerates attributable exposure completely, and reports terminal only when every order has positive terminal evidence. The existing scheduled-tick guard remains defense in depth, not the primary serialization boundary.

Reference generation and public/private request scheduling must be serialized independently. A time-derived value alone is not durable uniqueness.

## Amount and exposure model

- Load and validate target symbol precision, minimums, and fee rules before enabling writes.
- Convert price, volume, quote notional, and fees to explicit decimal or integer-scaled values; do not use binary floating point for accounting.
- Quantize before minimum, balance, and cap checks. Persist exactly the transmitted values.
- Enforce independent per-order, batch, inventory, daily-loss, and unresolved-outcome caps in the controller. Strategy parameters may only reduce, never bypass, those limits.
- Treat all open and outcome-unknown orders as liabilities/exposure until resolved.

## Required telemetry

Expose operator-visible held reason, age, affected intent/order IDs, reserved base/quote exposure, last complete scan, and reconciliation health. Redact login token, signing secret, control capability, private headers, and full error payloads. A health endpoint must distinguish process liveness, read freshness, reconciliation completeness, and write enablement.
