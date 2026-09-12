# 04 — Operator and admin screens

Designed for one person running operations from a phone, with a desktop version that adds density,
not features. Same design language as the customer side; the operator side is allowed precise
language ("screening", "risk band", "policy version") because the audience is trained.

The organising principle is **who acts next**. The operator home is not a list of deposits — it is a
list of things waiting on *them*, with everything else deliberately out of the way.

## O1 — Home / Action queue

```
┌─────────────────────────────────┐
│ CleanWallet        ⚙  ●3        │  ● = unread alerts
├─────────────────────────────────┤
│ ⚠ Screening provider degraded   │  system banner, amber, dismissible
│   since 14:02 · 3 deposits held │  per session, never auto-hidden
├─────────────────────────────────┤
│                                 │
│  Waiting on you            3    │  H1 + count, navy
│                                 │
│  ┌───────────────────────────┐  │
│  │ ● MATERIAL RISK           │  │  left border 4 px, red
│  │ 10,000 USDT · CW-98234    │  │
│  │ Ali H. · 12 min           │  │  ageing, red past SLA
│  │ Funds at intake · not     │  │  money truth, always visible
│  │ swept                     │  │
│  │              [ Review → ] │  │
│  └───────────────────────────┘  │
│                                 │
│  ┌───────────────────────────┐  │
│  │ ● NEEDS REVIEW            │  │  amber
│  │ 4,500 USDT · CW-98231     │  │
│  │ Evidence missing · 2 h    │  │
│  │              [ Review → ] │  │
│  └───────────────────────────┘  │
│                                 │
│  ┌───────────────────────────┐  │
│  │ ✓ REQUIREMENTS SATISFIED  │  │  green
│  │ 2,000 USDT · CW-98229     │  │
│  │ Ready to approve · 4 min  │  │
│  │            [ Approve → ]  │  │
│  └───────────────────────────┘  │
│                                 │
│  Waiting on others         7 ›  │  collapsed section
│  Customer 3 · Network 2 ·       │
│  Driver 1 · Provider 1          │
│                                 │
│  Today                          │
│  6 completed · 62,400 USDT      │
│  1 declined · 0 unresolved      │
└─────────────────────────────────┘
```

Rules:
- **Sorted by risk band, then by age.** Never by amount — a small deposit with a sanctions hit
  outranks a large clean one.
- Every card states the funds' real location (`Funds at intake`, `Swept`, `Payout authorized`). An
  operator must never have to open a case to learn whether money moved.
- `Approve` on the home card is a two-step action, not a single tap: it opens the decision sheet
  (O2's decision panel) with reason codes. There is no one-tap approve anywhere in the product.
- Counts are live. Zero state: "Nothing waiting on you. 7 items are with customers, the network, or
  a driver." — plus a `View all` link. Never a congratulatory empty state on a compliance queue.

## O2 — Deposit / case detail

The single most important screen. Six stacked sections on mobile, two columns at 1024 px.

**1. Money truth** (always first, never collapsed)

| Field | Example | Note |
| --- | --- | --- |
| Received | 10,000.00 USDT | Exact, from chain |
| Expected | 10,000.00 USDT | From order; delta shown if any |
| Network | Tron (TRC-20) | Contract address shown on tap |
| Confirmations | 20 / 20 · **Final** | Live; `Not final` is red |
| Transaction | `a3f9…c21b` | Copy + explorer link |
| Intake address | `TQn9…qXvY` · **Ownership proven 4 Mar** | Red if unproven — blocks approval |
| Funds location | **At intake. Not swept.** | The one line that stops mistakes |

**2. Order and customer match** — order ref, customer, KYC status, past order count, total volume,
and any prior cases. Repeat-customer context prevents the most common false positive.

**3. Screening result** — provider, band, score, obtained-at, reasons as chips, related addresses,
exposure breakdown, and the raw response behind a `View raw` disclosure. Stale results (older than
policy TTL) render with a `Re-screen` action rather than silently being trusted.

**4. Evidence checklist** — required items from the active policy, each `Received` / `Missing`, with
uploader and timestamp. Missing items expose one action: `Request from customer` (drafts the C3
request, operator reviews wording, sends).

**5. Decision panel**

```
Decision                                  CleanWallet recommends: HOLD
                                          Policy v12 · evaluated 14:06

( ) Approve — requirements satisfied
( ) Request information from customer
( ) Decline — no payout
( ) Close unresolved — retain evidence

Reason code            [ select ▾ ]       required, structured
Rationale              [ textarea  ]      required, min 20 chars

Funds stay at the intake address until you sign a sweep with your own
wallet. CleanWallet cannot move funds.                      (static note)

[ Record decision ]                        primary, disabled until valid
```

- The recommendation and the decision are stored separately. Operator override of a recommendation is
  allowed, always logged, and surfaced in the audit pack — that is the "you stay in control" promise
  with an accountability trail attached.
- `Approve` is blocked, with an inline explanation, when: deposit is not final; intake ownership is
  unproven; a required screening result is missing or degraded; the amount is outside tolerance
  without a mismatch resolution; or the case needs a second approver above threshold.
- Above the four-eyes threshold in solo-operator mode, the panel shows: "This amount requires a
  second approver. You can record your decision now — it will be marked *single approver* in the
  audit record." Honest, not blocking, and visible forever.

**6. Timeline** — append-only, newest last, every entry with actor (person, system, or provider),
timestamp, and hash prefix. Nothing in the UI can edit or delete an entry. Notes are added as new
entries.

**Sweep and payout blocks** appear after approval: `Register sweep transaction` (hash input,
validated against source/destination/amount/finality, with a clear `Mismatch` result state), then
`Authorize payout` (method, amount, recipient confirmation), then the acknowledgment record.

**States:** loading (money-truth skeleton first), not-found, no-screening-yet, provider-degraded
(inline, with what is being waited on), reversed deposit (full-width red banner, assessment shown as
`VOIDED`, all actions disabled except `Open incident`), already-decided (read-only with
`Supersede assessment` as the only path), permission-denied for read-only roles.

## O3 — Case workspace (review and alert)

Adds to O2: a structured review checklist tied to the reason codes, an RFI thread with the customer,
internal notes visible only to the dealer's staff, escalation to a named reviewer, a
`Report to authority` action that records that a report was filed (reference number, date, filer)
without storing the report's content, and a decision recorder with the four outcomes.

Two hard behaviours: `unresolved` requires a rationale and a retention date and is never auto-closed;
and a `Return funds` request is not an action in this product — it produces a task with a warning
("Returning funds to a source with a sanctions or stolen-funds signal may itself be an offence. This
requires a documented legal decision.") and a required legal-sign-off field before the operator can
mark it done outside the system.

## O4 — Addresses and wallets

Registry of dealer-owned addresses with `label`, `chain`, `purpose` (intake pool / treasury / fee),
`ownership proof` (method, proven-at, re-prove action), `status`, and current balance from the
indexer. Unregistered destinations are the alert surface: any outflow from an intake address to an
address not in this registry raises an `unexpected_outflow` alert within one block, with a
full-bleed banner on home. CleanWallet cannot block it; it must never fail to notice it.

Intake allocation is shown as a pool with a per-order assignment log and a strict never-reuse rule,
including a visible count of addresses remaining before the pool needs extending.

## O5 — Policy and rules

Versioned, immutable-once-published editor: tolerance bands, finality thresholds per chain,
risk-band thresholds, required evidence per band and amount, screening provider order and fallback,
outage tolerance, four-eyes threshold, retention periods, and order expiry.

Three behaviours that make this trustworthy: **effective dates** (never retroactive), **dry-run**
("this change would have moved 7 of the last 100 deposits from satisfied to review — see them"), and
**a diff view with an approver** on publish. Draft, review, publish — not live-editing a running rule
set.

## O6 — Reports and exports

- **Audit pack (per case)** — PDF + JSON: order, deposit facts, screening snapshot, policy version,
  decision with reason and approver, sweep verification, payout acknowledgment, full hash-chained
  timeline, and the standing disclaimer that a screening pass is not a guarantee of lawful funds.
- **Period exports** — CSV/JSON by date, band, outcome, customer, asset.
- **Exception register** — every override, every single-approver decision, every degraded-mode
  approval path, every unresolved case. This is the first thing an examiner asks for; it should take
  one tap.
- Every export is itself an event: who exported what, when, and with which filters.

## O7 — Security centre

Roles (`owner`, `operator`, `reviewer`, `auditor` read-only, `driver` scoped to assigned payouts),
passkey/2FA enforcement, session and device list with revoke, API keys with scopes and rotation,
webhook endpoints with signing secrets and delivery history, IP allowlisting, provider status, and a
full admin action log. High-risk settings (policy publish, role change, address registry change,
export of personal data) are step-up authenticated and, above threshold, dual-controlled.

## O8 — Mobile admin specifics

- Bottom tab bar: Queue · Cases · Wallets · More. Decision actions are reachable one-handed.
- Bottom sheets for decisions, never modals that hide the money-truth panel.
- Destructive and irreversible actions require a deliberate confirm with the amount typed or a
  slide-to-confirm — no accidental approvals in a pocket.
- Offline: read-only cached case view with a clear "Last updated 14:06 · offline" stamp. **No
  decision can be recorded offline.** Queued write actions are rejected, not buffered.
- Push notifications carry no customer PII and no screening outcome in the preview text.
