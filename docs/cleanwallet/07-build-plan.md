# 07 — Build plan

Scope discipline for launch: **one chain, one asset, one payout method, one dealer.** USDT on Tron,
cash delivery, the founder as sole operator. Every extra chain multiplies the finality, reorg, and
fee-model work; every extra payout rail multiplies the acknowledgment work. Breadth is P1.

## Stack

| Layer | Choice | Why |
| --- | --- | --- |
| Database | Postgres (Supabase) with RLS | Constraints and triggers carry the integrity rules; RLS gives per-dealer isolation and a read-only auditor role for free |
| Public API | TypeScript, serverless functions | Matches the existing Vercel deployment; stateless, no chain or provider credentials |
| Chain watcher | Long-running Node worker (not serverless) | Needs persistent connections, ordered processing, and confirmation tracking |
| Screening worker | Separate long-running worker | Isolates provider credentials from the order book |
| Outbox dispatcher | Worker reading committed events | Single outbound path, so "what did we tell the customer" is answerable |
| Customer UI | Server-rendered, minimal JS, progressive enhancement | A payment page must work on a weak connection and a cheap phone |
| Operator UI | SPA with server-driven state | Density, live queue, offline read-only |
| Files | Object storage, per-dealer keys, signed URLs | Evidence documents |

Two RPC sources per chain with a disagreement alarm. A single RPC provider is a single point of truth
about money, which is the one thing that must never have a single point.

## P0 — launch-blocking

Nothing ships without all of it. Grouped into six milestones, each independently demoable.

### M1 — Foundations and address integrity
- Tenancy, users, roles (`owner`, `operator`, `reviewer`, `auditor`, `driver`), passkey/2FA.
- Address registry with **ownership proof** (signed message + micro-transfer), re-proof, and the
  `intake_ownership_unproven` guard. *(Closes A1)*
- Append-only `events` table with trigger-computed hash chain; `UPDATE`/`DELETE` revoked. *(B5)*
- Policy v1 as a versioned document — editor comes later, but the version is stored from day one. *(B1)*

**Demo:** register an address, fail to create an order against it, prove ownership, succeed. Show an
attempt to edit an event failing at the database.

### M2 — Orders and detection
- Order creation with intake allocation, never-reused, one address per order.
- Chain watcher for Tron: transfer detection at 0 conf, confirmation tracking, finality at the
  configured threshold, reorg detection and `deposit.reversed`. *(A2)*
- Money key uniqueness and a daily chain-vs-credited reconciliation job. *(A6)*
- Full mismatch matrix: underpay, overpay, multiple, late, wrong asset, dust, zero-value. *(A3, A4, A5)*

**Demo:** on testnet — exact payment, underpayment then top-up, overpayment, a late payment on an
expired order, a wrong-token transfer, and a forced reorg that voids an assessment.

### M3 — Screening and assessment
- Provider integration with raw-response snapshot and hash. *(B2)*
- Degraded mode: hold, never pass; outage recording; auto re-screen on recovery. *(B6)*
- Policy engine producing `satisfied` / `needs_review` / `material_risk` with reasons, storing
  `policy_version` and `inputs_sha256`. *(B1)*
- Decision recording with mandatory reason codes and rationale; recommendation stored separately
  from decision. *(B3)*

**Demo:** three deposits producing the three outcomes; kill the provider mid-flight and show the
assessment holding with no override available.

### M4 — Customer experience
- C1 Pay screen with QR, per-order address, expiry, optional hash assist.
- C2 Tracker with all six states, mismatch cards, and progressive honesty on timing. *(C1)*
- Neutral hold copy with a CI lint rule blocking the banned-words list. *(C2, T10)*
- C5 Receipt, PDF export. *(C4)*
- Notifications for the P0 subset: detected, final, info required, authorized, acknowledged,
  mismatch, reversed.

**Demo:** the full customer journey on a 375 px phone, throttled network, in every state.

### M5 — Operator experience
- O1 Home grouped by `awaiting_party`, sorted risk-then-age, with the funds-location line. *(C6)*
- O2 Case detail with the money-truth panel, screening result, evidence checklist, decision panel
  with all five approval guards, and the append-only timeline.
- Sweep registration and verification: source, destination allowlist, asset, amount, finality;
  `sweep.mismatch` blocks payout. *(A8)*
- Payout authorization and acknowledgment with handover code. *(A9)*
- Audit pack export (PDF + JSON) with the hash chain and the standing disclaimer.

**Demo:** approve a clean deposit end to end from a phone; then attempt each blocked approval and
show the specific reason.

### M6 — Launch gates
- Restore test from backup; reconciliation job green for seven consecutive days on testnet.
- Security review of the four services and the RLS policies; secret scanning clean.
- The three sign-offs in [`06-security-and-integrity.md`](06-security-and-integrity.md): counsel,
  provider terms, and one audit pack reviewed by a real auditor.
- Public copy audit: every claim on the marketing site traceable to something the product does.

## P1 — first 90 days

| Item | Closes |
| --- | --- |
| `outflow.unexpected` detection and alerting | A7 |
| Four-eyes above threshold + honest solo-operator labelling | B4 |
| SLA clock, ageing, breach notifications | B7 |
| Request-for-information flow with uploads and checklist | C3 |
| Full notification matrix incl. SMS and driver ETA | C5 |
| Policy editor: draft, diff, dry-run, publish with approver | B1 |
| Dealer-facing API keys, webhooks, delivery history | — |
| Exception register and period exports | — |
| Case workspace: escalation, report-reference recording, unresolved retention | — |
| Second chain (Ethereum/USDT) and second payout rail (bank transfer) | — |
| Driver surface: assigned payouts, handover code entry, proof of delivery | — |

## Polish

Saved views and batch actions; operator analytics (decision times, override rate, band mix);
print-perfect audit packs; Arabic and French localisation with full RTL layout; onboarding wizard for
a new dealer including guided address proof; dark mode; multi-dealer white-label; a status page — only
once uptime is actually measured.

## Acceptance criteria

Written as tests. A milestone is done when every criterion has a named, passing test.

**Address integrity**
- Creating an order against an unproven intake address returns `409 intake_ownership_unproven`.
- An intake address already bound to an order cannot be allocated again (`409 address_already_assigned`).
- Ownership proof by signed message fails closed on a signature from a different key.

**Detection and finality**
- A deposit at `confirmations < finality_confirmations` cannot receive an approved decision
  (`409 deposit_not_final`).
- Replaying the same `(chain, tx_hash, output_index, asset)` creates no second deposit row.
- A simulated reorg removing a `final` deposit sets `reversed`, voids the assessment, raises a P1
  alert, and leaves every prior event intact.
- A payment 6% below expectation with a 1% tolerance lands in `mismatch_hold` with the exact figures
  in the customer copy.
- A payment on an expired order lands in `mismatch_hold`, not `expired`, and is never orphaned.
- A non-configured token transfer lands in `unsupported_asset` and never appears as a payment.

**Screening and decisions**
- With the provider returning 500, the assessment is `needs_review` with reason
  `screening_unavailable`, and no code path — API, UI, or admin — can produce `satisfied`.
- A decision without a reason code returns `400 reason_code_required`.
- A decision that overrides the recommendation is stored with both values and appears in the
  exception register.
- Two concurrent `Approve` requests with the same idempotency key create exactly one decision.
- Above the four-eyes threshold in solo mode, the decision records `approver_mode = single_approver`
  and the audit pack shows it.

**Settlement**
- A sweep to an unregistered destination returns `422 sweep_destination_unregistered` and blocks
  payout.
- `POST /v1/payouts` without a verified sweep returns `409 no_verified_sweep`.
- A payout closes only on a recorded acknowledgment.

**Evidence**
- `UPDATE` and `DELETE` on `events` fail for the application role.
- Recomputing the hash chain over an exported pack reproduces the stored `chain_sha256`.
- An audit pack for an `unresolved` case exports successfully and contains the screening snapshot,
  policy version, decision rationale, and the disclaimer.
- Any transfer-shaped API request returns `501 transfer_not_supported`.

**Experience**
- Every customer-facing string passes the banned-words lint.
- Every screen renders correctly at 375 px with no horizontal scroll, and in loading, empty, error,
  pending, success, cancelled, failed, and review states.
- No push notification payload contains a name, a document, a risk band, or a screening reason.
- The operator home, offline, shows cached cases read-only and rejects a decision attempt.

## Definition of done, per feature

Code, database constraints, tests named after the acceptance criteria, copy reviewed against the
banned-words list, all eight UI states implemented, an entry in the events taxonomy, a webhook event
if state changed, audit-pack coverage, and a line in the release note saying what a dealer can now
rely on — and what they still cannot.
