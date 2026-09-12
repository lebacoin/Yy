# 02 — Flow, states, and who acts next

## Actors

| Actor | Role | Holds keys? |
| --- | --- | --- |
| **Customer** | Sells crypto, expects the agreed payout (cash, driver delivery, or transfer) | Their own wallet |
| **Dealer** | The CleanWallet tenant. Owns the intake address and the treasury | Yes — all of them |
| **Operator** | The person on the dealer side who decides and authorizes payout (often the founder) | Via the dealer's own wallet tooling, never in CleanWallet |
| **Reviewer** | Second pair of eyes on cases above threshold | No |
| **Driver** | Delivers cash and captures acknowledgment | No |
| **CleanWallet** | Reads chains, screens, assesses, records | **No. Ever.** |
| **Blockchain** | Confirms and finalises | n/a |
| **Screening provider** | KYT / sanctions / path analysis | n/a |

## The flow

```
1  ORDER              Operator creates an order: asset, network, expected amount, payout method,
                      customer ref, expiry. CleanWallet allocates one dealer-owned intake address
                      to that order — never reused, ownership already proven.

2  CUSTOMER PAYS      Customer sends crypto to the intake address. CleanWallet shows a live
                      tracker. Transaction hash is an optional assist, never a requirement.

3  DETECT & VERIFY    Watcher sees the transfer at 0 confirmations and says so immediately.
                      Verifies asset contract, network, amount against tolerance, sender set.
                      Waits for the chain's finality threshold before any assessment can pass.

4  SCREEN             KYT / sanctions / path analysis on sender addresses and the transaction.
                      Raw provider response stored and hashed. Provider down => degraded, never pass.

5  ASSESS             Policy engine (versioned) produces one of three outcomes:
                        REQUIREMENTS SATISFIED  — evidence and policy checks pass
                        NEEDS REVIEW            — evidence missing, or a required service unavailable
                        MATERIAL RISK ALERT     — documented human review, case opened
                      CleanWallet recommends. The operator decides. Both are recorded.

6  DEALER SIGNS SWEEP Only on an approved assessment, and only by the dealer, with the dealer's keys,
                      outside CleanWallet. The operator registers the sweep transaction hash.

7  VERIFY SWEEP       from = order intake address, to = registered treasury address, asset and amount
                      reconcile, finality reached. Anything else is a mismatch alert.

8  AUTHORIZE PAYOUT   Operator authorizes the agreed payout. CleanWallet records the authorization
                      and then the acknowledgment (customer confirmation, or driver proof of delivery).
                      CleanWallet does not move fiat and has no payout rail.

9  RECORD             Every step above is one append-only, hash-chained event. Case closes with an
                      outcome code. Unresolved cases are retained, not deleted.
```

On **NEEDS REVIEW** and **MATERIAL RISK ALERT** the funds stay at the intake address. There is no
sweep, no payout, and no automatic refund, under any policy setting. This is not configurable.

## State machines

### Order

`draft → awaiting_payment → payment_detected → verifying → assessing → decision_pending → approved → swept → payout_authorized → completed`

Terminal side-exits: `expired`, `cancelled`, `declined`, `closed_unresolved`.

| State | Meaning | Who acts next |
| --- | --- | --- |
| `draft` | Created, address not yet allocated | Lebacoin/dealer |
| `awaiting_payment` | Address live, nothing seen | Customer |
| `payment_detected` | Seen in mempool or at 0 conf | Blockchain |
| `verifying` | Confirming toward finality | Blockchain |
| `assessing` | Screening and policy running | Lebacoin/dealer |
| `decision_pending` | Assessment ready, awaiting operator | Lebacoin/dealer |
| `approved` | Operator approved; sweep not yet signed | Lebacoin/dealer |
| `swept` | Sweep verified on-chain | Lebacoin/dealer |
| `payout_authorized` | Payout released, acknowledgment pending | Driver or customer |
| `completed` | Acknowledged, case closed | Nobody |
| `mismatch_hold` | Under/over/late/wrong-asset — see matrix | Customer or dealer |
| `info_required` | Request for information open | Customer |
| `in_review` | Case open, human review | Reviewer |
| `expired` | No payment before expiry | Nobody (re-quote to reopen) |
| `cancelled` | Cancelled before payment | Nobody |
| `declined` | Decided against payout; funds remain at intake | Support |
| `closed_unresolved` | No safe conclusion reached; evidence retained | Support |

### Deposit (on-chain fact, independent of the order)

`detected → confirming → final → matched` with `reversed` reachable from `detected`, `confirming`,
and `final` (deep reorg), and `unmatched` / `unsupported_asset` as holding states.

`reversed` is the dangerous one. Rules:
- A deposit below its chain's finality threshold can never carry an approved assessment.
- If a `final` deposit reverses, the assessment is **voided**, the order returns to `mismatch_hold`,
  and a `P1` incident is raised — regardless of whether payout already happened.
- If payout already happened, the case closes as `loss_event` with full evidence. The product does
  not pretend this is recoverable; it makes it visible and rare (finality thresholds exist for this).

### Assessment

`pending → satisfied | needs_review | material_risk` then `recorded` (operator decision attached),
with `voided` on deposit reversal or policy recall. Assessments are immutable once recorded; a new
assessment supersedes rather than edits, and the supersession chain is part of the record.

### Case

`open → gathering_evidence → escalated → decided → closed` with outcomes
`approved_after_review`, `declined`, `unresolved`, `reported_to_authority`.
`unresolved` is a first-class, retained outcome. Never auto-close on age.

### Sweep

`registered → verifying → verified` or `mismatch` (wrong source, wrong destination, amount
reconciliation failure, or asset mismatch). `mismatch` blocks payout authorization.

### Payout

`authorized → in_transit → acknowledged` or `failed` / `returned`. `in_transit` applies to driver
delivery and carries the driver reference and ETA.

## Who acts next

Every open item in the system carries exactly one `awaiting_party` from this enum:

`customer` · `dealer` · `blockchain` · `screening_provider` · `driver` · `support` · `reviewer` · `none`

It is not a cosmetic label. It is the operator home's primary grouping, it drives the SLA clock, and
it drives customer-facing copy. "Waiting on the blockchain" and "waiting on us" must never look the
same to anyone.

| awaiting_party | Customer sees | Operator sees | Clock |
| --- | --- | --- | --- |
| `customer` | "Action needed" + one CTA | "Waiting on customer" | Paused after reminder |
| `dealer` | "We're reviewing your payment" | **Your queue. Top of home.** | Running, ageing visible |
| `blockchain` | "Confirming on the network — 12 of 20" | "Confirming" | Running, expected-by shown |
| `screening_provider` | "Verification in progress" | "Provider degraded" + banner | Running, escalate at 30 min |
| `driver` | "On the way" + ETA | "Out for delivery" | Running |
| `support` | "Our team will contact you" | "Support queue" | Running |
| `reviewer` | "Verification in progress" | "Awaiting second approver" | Running |

## Payment-mismatch matrix

| Situation | Detection | State | Customer copy (exact) | Operator action |
| --- | --- | --- | --- | --- |
| Exact amount | Within tolerance | continue | — | — |
| Underpaid within tolerance | `amount < expected`, gap ≤ policy tolerance | continue, flagged | — | Accepted automatically, shown on the case |
| Underpaid beyond tolerance | gap > tolerance | `mismatch_hold` | "We received 9,400 USDT. Your order is for 10,000 USDT. Send the remaining 600 USDT to the same address, or ask us to re-quote." | Top-up, re-quote, or decline |
| Overpaid | `amount > expected` + tolerance | `mismatch_hold` | "We received 10,500 USDT — 500 more than your order. Choose: increase the payout, or we return the extra after review." | Extend order, or record a return decision (never automatic) |
| Two payments, one order | Two deposits matched to one order | `mismatch_hold` then aggregate | "We received two payments for this order. We're combining them now." | Aggregate or split into a second order |
| Late payment (after expiry) | Deposit on an expired order's address | `mismatch_hold` | "This payment arrived after the order expired. Your funds are safe. We'll re-quote at today's rate." | Re-quote and relink — funds are never orphaned |
| Wrong network | Asset/contract or chain mismatch | `unsupported_asset` | "This payment arrived on a different network than the order. Our team is checking whether it can be recovered. We'll tell you either way." | Recoverability assessment, honest outcome |
| Unsupported token / dust | Unknown contract on intake address | `unsupported_asset`, hidden by default | — (not shown unless customer-linked) | Ignore, or open a recovery case |
| Zero-value / spoof transfer | Value 0, or lookalike-address dust | Auto-ignored, logged | — | None. Never surfaced as a payment |

## Degraded mode

When a required screening provider fails, times out, or returns a partial result:

1. The assessment becomes `needs_review` with reason `screening_unavailable`. **It never becomes
   satisfied.** No policy setting, no operator override, no "expedite" button changes this.
2. `awaiting_party = screening_provider`; the operator home shows a system banner naming the
   provider and the outage start time.
3. Customer sees "Verification in progress" with an expected-by time — not an error, not a promise.
4. On recovery, queued deposits re-screen automatically, oldest first, and the outage window is
   written into every affected case's evidence.
5. If the outage exceeds the policy's tolerance (default 4 hours), every affected case escalates to
   the operator with a recommendation to hold, and the customer gets a proactive update.

The one legitimate override: a dealer policy may allow approval on a *secondary* provider's result
when the primary is down, if the secondary is configured and its result is stored like any other.
That is a provider fallback, not an outage bypass, and the audit pack shows which provider ran.

## Notification matrix

| Trigger | Customer | Operator |
| --- | --- | --- |
| Payment detected (0 conf) | Push + in-app | In-app |
| Deposit final | Push | In-app |
| Assessment ready | — | **Push** (this is their queue) |
| Info required | Push + email + SMS | — |
| Approved / payout authorized | Push + email receipt | — |
| Driver dispatched | Push + SMS with ETA | In-app |
| Acknowledged / completed | Push + email receipt | In-app |
| Mismatch hold | Push + email with the exact number | **Push** |
| Deposit reversed | Push (neutral) | **Push, P1** |
| Unexpected outflow from intake | — | **Push, P1** |
| Sweep mismatch | — | **Push, P1** |
| Provider degraded > 30 min | Proactive update at 1 h | **Push** |
| SLA breach on dealer queue | — | **Push** |

Customers never receive a notification that reveals a screening result, a risk band, a provider
name, or the existence of a report to an authority.
