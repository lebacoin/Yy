# 08 — Implementation handoff

Ticket-by-ticket spec. Each ticket is independently reviewable, names the gap it closes from
[`01-audit.md`](01-audit.md), and ends with the test that proves it. Build in order — later tickets
assume the guards from earlier ones.

Global rules for every ticket:

- Money is `numeric(38,0)` in smallest units plus `decimals`. No floats. No money in a display string.
- Integrity rules live in the database first (constraints, triggers, RLS), then in code as guard
  clauses with the documented error codes, then in tests named after the acceptance criterion.
- Every state change writes exactly one `events` row through the append-only path, and emits a
  webhook if a dealer-visible state changed.
- Every customer-facing string goes through the copy lint. Banned on customer surfaces: `AML`,
  `KYT`, `sanctions`, `risk`, `score`, `flag`, `flagged`, `suspicious`, `investigation`,
  `blocked`, `quarantine`, `sweep`, `illicit`, `laundering`.
- No ticket introduces a code path that signs, transfers, refunds, or returns funds.

---

### T1 — Schema, append-only events, RLS
**Closes:** B5, and the foundation for everything.
Create the schema in [`05-data-model-and-api.md`](05-data-model-and-api.md). Implement `events` with
a `BEFORE INSERT` trigger computing `payload_sha256` and
`chain_sha256 = sha256(prev_sha256 || payload_sha256)` under a per-dealer advisory lock so the chain
is strictly ordered. Revoke `UPDATE`, `DELETE` on `events` from the application role. Enable RLS on
every table keyed by `dealer_id`; `auditor` gets `SELECT` only; `driver` is scoped to assigned
`payouts` and cannot reach `documents`, `screenings`, or `cases`.
**Test:** app role `UPDATE events` fails; a chain recomputation over 1,000 inserted events matches
the stored head; an `auditor` session cannot insert; a `driver` session selecting `screenings`
returns zero rows.

### T2 — Auth, roles, step-up
Passkeys (WebAuthn) with TOTP fallback; mandatory for `owner` and `operator`. Step-up
re-authentication required for: recording a decision, publishing a policy, changing the address
registry, creating or rotating an API key, and exporting personal data. Session and device list with
revoke. API keys: `sk_test_` / `sk_live_`, stored hashed, scoped, rotatable.
**Test:** a decision request with a stale step-up token returns 401; a revoked session cannot act; a
key without the `decisions:write` scope is rejected.

### T3 — Address registry and ownership proof
**Closes:** A1.
`POST /v1/addresses` registers a dealer-owned address with `purpose`. Ownership proof by one of:
(a) **signed message** — server issues a nonce, dealer signs it with the address key, server verifies
against the address; (b) **micro-transfer** — an inbound transfer from an already-proven dealer
address, matched by amount and memo; (c) **descriptor** — an xpub/descriptor whose derived addresses
are accepted, proof applying to the whole branch. Store method, reference, and `ownership_proven_at`.
Re-proof required if a descriptor changes. Unproven addresses cannot be allocated to orders.
**Test:** a signature from a different key is rejected; order creation against an unproven address
returns `409 intake_ownership_unproven`; a proven descriptor's derived address is allocatable without
a separate proof.

### T4 — Orders and intake allocation
`POST /v1/orders` per the example payload. Allocate one proven, unused intake address; enforce
`UNIQUE (intake_address_id)` so reuse is impossible at the database level. Store `expires_at`,
`payout_method`, `awaiting_party = customer`, and the active `policy_id`. Implement `cancel` and
`requote` (requote creates a new rate on the same order and relinks any late deposit).
**Test:** two concurrent order creations cannot receive the same intake address; a cancelled order
rejects new deposits into `mismatch_hold` rather than crediting them.

### T5 — Tron chain watcher
**Closes:** A2 (part), A6 (part).
Long-running worker. Two independent RPC/indexer sources with a disagreement alarm. Detect TRC-20
transfers to intake addresses: emit `deposit.detected` at first sight, track `confirmations`, emit
`deposit.final` at `assets.finality_confirmations`, and re-verify `final` deposits for a trailing
window so a deep reorg is caught. On reversal: set `reversed`, emit `deposit.reversed`, void any
assessment, raise a P1 alert. Never write a decision. Ignore zero-value transfers and log them.
**Test:** on testnet, detection latency under 15 s; a forced reorg voids the assessment and raises
the alert; a spoofed lookalike-contract transfer is not recorded as USDT; the two sources
disagreeing raises the alarm rather than picking one.

### T6 — Deposit matching and the mismatch matrix
**Closes:** A3, A4, A5, A6.
Match deposits to orders by intake address. Enforce
`UNIQUE (chain, tx_hash, output_index, asset_id)`. Implement every row of the mismatch matrix in
[`02-flow-and-states.md`](02-flow-and-states.md) with its own state, reason, and customer copy:
tolerance accept, underpay hold with top-up, overpay hold, multi-payment aggregation, late payment
relink, wrong network `unsupported_asset`, unknown token hidden, zero-value ignored. Daily
reconciliation comparing on-chain totals per intake address against credited totals, alerting on any
delta.
**Test:** each matrix row has a named test asserting state, `awaiting_party`, and the exact customer
string; replaying a deposit creates no second row; reconciliation detects an injected discrepancy.

### T7 — Screening integration and degraded mode
**Closes:** B2, B6.
Provider abstraction with a primary and an optional configured fallback. Store `raw_response` plus
`raw_response_sha256`, `band`, `score`, `reasons`, `related_addresses`, `ttl_expires_at`. On failure,
timeout, or partial result: mark `degraded`, open a `provider_outages` row, hold the assessment, set
`awaiting_party = screening_provider`, emit `screening.degraded`. Auto re-screen oldest-first on
recovery and write the outage window into affected cases. Rate-limited manual `rescreen`.
**Test:** provider 500 yields `needs_review` with `screening_unavailable` and no reachable code path
produces `satisfied`; a result past its TTL is not reused for approval; the fallback provider's
identity appears in the stored assessment.

### T8 — Policy engine and versioned policies
**Closes:** B1.
Policy document: tolerance bps, finality per chain, band thresholds, required evidence per band and
amount, provider order and fallback, outage tolerance, four-eyes threshold, retention periods, order
expiry. Immutable once published, `effective_from` never retroactive, publish requires an approver
and stores a diff. Evaluation produces `outcome`, `recommendation`, `reasons`, and persists
`policy_id` + `inputs_sha256`.
**Test:** re-evaluating an old deposit under its stored policy version reproduces the original
outcome bit-for-bit; publishing without an approver is rejected; an `effective_from` in the past is
rejected.

### T9 — Decision recording and approval guards
**Closes:** B3, B4.
`POST /v1/assessments/{id}/decision` with `action`, `reason_code`, `rationale` (min 20 chars).
Idempotent on `Idempotency-Key`; partial unique index allows one non-voided assessment per deposit.
Guards, each with its documented error code: not final, ownership unproven, screening
missing/degraded, unresolved mismatch, second approver required. Store `recommendation` and the
decision separately. `approver_mode` is `single_approver` when solo above threshold, and that value
is rendered in the audit pack.
**Test:** every guard has a test asserting its specific error code; two concurrent approvals create
one decision; an override is retrievable from the exception register.

### T10 — Customer pay and tracker screens
**Closes:** C1, C2.
Server-rendered, progressive enhancement, 375 px baseline. C1 per
[`03-screens-customer.md`](03-screens-customer.md): QR, full address, copy, countdown, network
warning, single CTA, optional hash sheet. C2: four-step stepper, six states, mismatch cards with
exact figures, progressive honesty on timing, live region announcements, reduced-motion support.
Money figures render only from server-confirmed values.
**Test:** all six states plus loading/error/expired/cancelled/offline render at 375 px with no
horizontal scroll; copy lint passes; AA contrast verified; no screening outcome appears in any
rendered HTML or JSON reaching the client.

### T11 — Operator home and case detail
**Closes:** C6.
O1 grouped by `awaiting_party`, sorted band-then-age, funds-location on every card, live counts,
degraded banner, honest zero state. O2 with the money-truth panel first, order/customer match,
screening result with raw disclosure, evidence checklist, decision panel, append-only timeline.
Mobile: bottom tabs, bottom sheets, slide-to-confirm on irreversible actions, offline read-only with
decisions rejected rather than queued.
**Test:** a case with an unproven address shows the approve guard inline with the reason; the
funds-location line is present on every card and case; an offline decision attempt is rejected with
a clear message; the timeline exposes no edit or delete affordance.

### T12 — Sweep verification
**Closes:** A8, and partially A7.
`POST /v1/sweeps` takes a dealer-signed transaction hash. Verify `from` = the order's intake address,
`to` ∈ registered treasury addresses, asset matches, amount reconciles against the deposit minus
fees, and finality reached. Set `verified` or `mismatch` with a reason; `mismatch` blocks payout.
Separately, watch intake addresses for any outflow to an unregistered destination and raise
`outflow.unexpected` within one block.
**Test:** a sweep to an unregistered address returns `422 sweep_destination_unregistered`; an
unexpected outflow raises the alert within one block on testnet and writes an event.

### T13 — Payout authorization and acknowledgment
**Closes:** A9.
`POST /v1/payouts` requires a verified sweep (`409 no_verified_sweep` otherwise). Methods:
`cash_delivery` (driver ref, ETA, handover code — store only a hash, compare on entry),
`cash_pickup`, `bank_transfer` (reference), `wallet_credit`. Acknowledgment closes the payout and the
case. Driver role is scoped to assigned payouts, sees no customer documents and no case data.
**Test:** payout without a verified sweep is rejected; a wrong handover code does not acknowledge; a
driver session cannot read a case; the order reaches `completed` only after acknowledgment.

### T14 — Audit pack and exception register
Per-case PDF + JSON containing order, deposit facts, screening snapshot with its hash, policy
version, decision with reason/rationale/approver/approver_mode, sweep verification, payout
acknowledgment, the full hash-chained timeline, and the standing disclaimer. Exception register
listing overrides, single-approver decisions, degraded-mode paths, and unresolved cases. Every export
writes an `exports` row.
**Test:** a pack for an `unresolved` case exports and validates; recomputing the chain from the pack
matches the stored head; the disclaimer is present in both formats; the exception register surfaces
an injected override.

### T15 — Notifications and outbox
Single outbound dispatcher reading committed events. P0 subset from the notification matrix. Payloads
carry no name, document, band, reason, or provider. Webhook delivery: `CleanWallet-Signature`
(`t=…,v1=…`), 24 h exponential backoff, monotonic `sequence` per order, at-least-once with
`event.id` for consumer idempotency.
**Test:** no notification payload contains PII or an outcome; a replayed webhook past the timestamp
window is rejected by the documented verification snippet; sequence numbers are monotonic per order
under concurrent state changes.

### T16 — Copy lint and claim audit
CI check over customer-facing strings for the banned-words list; a second check asserting the
disclaimer string is present in the API order response, the customer receipt, and the audit pack
template. Review every claim on the public site against what the product does, and add the
"proposed workflow, not a live product" status wherever CleanWallet is described before launch.
**Test:** introducing `"your payment was flagged"` into a customer template fails CI; removing the
disclaimer from any of the three surfaces fails CI.

---

## Build order and parallelisation

```
T1 ─ T2 ─ T3 ─ T4 ─┬─ T5 ─ T6 ─┬─ T7 ─ T8 ─ T9 ─┬─ T11 ─ T12 ─ T13 ─ T14
                   │           │                 │
                   └───────────┴─ T10 ───────────┘         T15, T16 run alongside from T5
```

T1 through T4 are strictly sequential — they are the integrity foundation. T5/T6 (chain truth) and
T7/T8 (screening truth) can proceed in parallel once T4 lands. T10 (customer UI) can start as soon as
T6's states exist. T15 and T16 are continuous from T5 onward.

## What "done" does not mean

Shipping T1–T16 produces a product that runs the configured checks and records them well. It does not
produce a compliance guarantee, a licence, or an assurance that funds are lawful. The three sign-offs
in [`06-security-and-integrity.md`](06-security-and-integrity.md) — counsel, provider terms, and a
real auditor's review of one audit pack — are launch gates, and no amount of code closes them.
