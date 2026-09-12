# 01 — Audit of the launch scheme

Two schemes were supplied. They are not two drafts of the same thing; the second one changes what
kind of company CleanWallet is. That distinction drives the whole plan, so it is recorded first.

## Decision 1 — the corrected scheme is the only buildable one

| | Scheme A ("How It Works") | Scheme B ("Corrected Launch Scheme") |
| --- | --- | --- |
| Deposit address | Provided by CleanWallet | Dealer-owned, order-linked |
| Private keys | CleanWallet's | Dealer's only |
| Sweep | CleanWallet auto-sweeps to treasury | Dealer signs; CleanWallet verifies the receipt |
| Medium risk | Funds moved to a CleanWallet quarantine wallet | Funds stay at intake; case opened |
| High risk | Funds moved to a CleanWallet blocked wallet, "return (optional)" | Funds stay at intake; documented human review; no automatic refund |
| What CleanWallet is | A custodian with a compliance screen | A compliance and evidence system of record |

**Scheme A is not shippable, and the problem is not UX.** Holding customer crypto in CleanWallet
sweep / quarantine / blocked wallets makes CleanWallet a custodian of third-party virtual assets.
In most regimes that is a licensed activity (VASP / MSB / money-transmission, with capital, audit,
safeguarding and reporting duties attached), and it moves the single largest operational risk in the
business — key compromise — onto a company that has not launched yet. The "return (optional)" arrow
is worse than unlicensed: automatically sending funds back to a sanctioned or stolen-funds source is
a potential sanctions breach, and doing it after a hit can be an offence in its own right.

**Scheme B removes the licensing surface and the key-custody risk in one move**, and it does it
without weakening the product, because the thing the dealer is actually buying is not custody. It is
*certainty before payout*. Rating below is therefore against Scheme B.

Scheme A survives in exactly one place: as a configuration option where the *dealer's own*
segregation addresses (their quarantine, their treasury) are registered with CleanWallet, and
CleanWallet only verifies and records movements between them. The wallets stay; the keys never move.

## Rating

**Scheme B as a launch scheme: 7.5 / 10.**
(Scheme A: 4 / 10 — good product instinct, unshippable risk model.)

What Scheme B gets right, and it is most of the hard part:

- Non-custodial by construction, stated on the page, not buried in terms.
- Read-only checks, so a CleanWallet compromise cannot move a customer's money.
- No automatic refund. Correct, and for the right reason.
- Funds stay at intake on both review and alert — one rule, no special cases.
- Evidence retained *including unresolved cases*. Most compliance tooling quietly drops the
  inconclusive ones, which are the only ones an examiner will ask about.
- "A sweep does not erase the source-of-funds record." That is a real engineering constraint
  written as a product promise.
- "A screening pass is not a guarantee of lawful funds." Honest. Keep this sentence exactly.

## What prevents 10/10

Twenty-two gaps, grouped. Every one of them is a state, a control, or a screen that does not exist
yet in the scheme. The plan in this folder closes all of them; the ID is referenced by the tickets
in [`08-codex-handoff.md`](08-codex-handoff.md).

### A. Money-truth gaps — wrong answers about real money

| ID | Gap | Why it breaks the deal |
| --- | --- | --- |
| A1 | **No intake-address ownership proof.** Nothing stops a dealer (or someone who compromised a dealer account) from registering an address they do not control. | CleanWallet would issue a clean assessment over a stranger's address, which is exactly the laundering pattern the product exists to stop. Fix: signed-message challenge, or an xpub/descriptor registration, before an address can accept an order. |
| A2 | **No finality or reorg policy.** "Confirmations" appears as a checked item, not as a threshold with a reversal path. | A deposit can un-confirm after an assessment. Payout is irreversible; the deposit was not. Fix: per-chain finality thresholds, `deposit.reversed`, and an assessment that is automatically voided on reversal. |
| A3 | **No underpay / overpay handling.** The scheme assumes one payment equals the order. | Real OTC: customer sends 9,980 USDT for a 10,000 order (their exchange took a fee). Silent accept loses the dealer money; silent reject loses the customer. Fix: tolerance bands, top-up flow, explicit overpay decision. |
| A4 | **No multi-payment or late-payment path.** | Two sends for one order, or funds arriving after expiry. Funds must never be orphaned, and never double-credited. |
| A5 | **No wrong-asset / wrong-network state.** USDT on Ethereum sent to a TRON address; a random token dusted onto the intake address. | Today this is an unhandled crash in the model. Fix: `unsupported_asset` state, honest recoverability statement, no promise to recover what cannot be recovered. |
| A6 | **No double-credit key.** | The same transaction matched to two orders pays out twice. Fix: unique constraint on `(chain, tx_hash, output_index, asset)` as the money key. |
| A7 | **No unexpected-outflow detection.** CleanWallet watches money in, nothing watches money out. | Funds leaving intake to an unregistered address is either a compromised dealer or a deal being settled outside the record. CleanWallet cannot stop it — it must detect and record it within one block. |
| A8 | **Sweep receipt is unspecified.** Step 5 says "verify the sweep receipt" without saying what makes one valid. | Fix: sweep is valid only if `from` = the order's intake address, `to` = a registered dealer treasury address, amount and asset reconcile, and it reaches finality. |
| A9 | **Payout acknowledgment is unproven.** The payout happens off-chain (cash, driver, transfer) and the scheme just says the operator authorizes it. | Without a recorded acknowledgment, "we paid" is an assertion. Fix: payout record with method, amount, authorizer, timestamp, and a customer-side or driver-side acknowledgment. |

### B. Decision-integrity gaps — right answer, unprovable later

| ID | Gap | Fix |
| --- | --- | --- |
| B1 | **No policy versioning.** A decision made under March rules is meaningless if the rules changed in April and nobody stored which version ran. | Immutable policy versions; every assessment stores `policy_version` and a hash of its inputs. |
| B2 | **Screening result is not snapshotted.** Providers re-score addresses; the score you saw is not the score the provider returns next year. | Store the raw provider response plus its hash. The record shows what was known at decision time. |
| B3 | **No reason codes on decisions.** "Dealer approves" is not an audit trail. | Mandatory structured reason code plus free-text rationale on every approve, decline, and unresolved close. |
| B4 | **No four-eyes, and no honest handling of its absence.** A one-person operation cannot do maker-checker. | Solo-operator mode: allowed, logged as `single_approver`, surfaced in the audit pack, with a threshold above which a second approver is required even in solo mode (use an external compliance reviewer). Never silently label a one-person approval as reviewed. |
| B5 | **Audit trail is not tamper-evident.** "Keep the evidence" does not say what stops the evidence being edited. | Append-only event table, hash-chained (`prev_hash` + `payload_hash`), periodic anchoring, exports carry the chain. |
| B6 | **No provider-outage doctrine.** Scheme B hints at it ("a required screening service is unavailable" → needs review) but does not forbid the dangerous default. | Hard rule: a screening outage can never produce an approval. Degraded mode holds and says so on both sides. |
| B7 | **No SLA or ownership clock.** Nothing says how long a case may sit. | `awaiting_party` + `sla_due_at` on every open item, with ageing visible on the operator home. |

### C. Experience gaps — the entire customer side is missing

| ID | Gap | Fix |
| --- | --- | --- |
| C1 | **The customer sees nothing.** Both schemes are operator-facing. The person who just sent 10,000 USDT and is waiting for cash has no screen. | Full customer flow in [`03-screens-customer.md`](03-screens-customer.md): pay, track, respond, receipt. |
| C2 | **No honest hold copy — and a tipping-off trap.** The obvious copy ("your funds were flagged for AML") can constitute tipping off, which is an offence in many regimes. The opposite ("almost done!") is a lie. | One neutral vocabulary, used on every hold, reviewed against a tipping-off rule: say that a check is in progress and who acts next, never the reason or the provider result. |
| C3 | **No evidence-request flow.** "Gather missing evidence" has no channel. | In-app request-for-information with a checklist, upload, and status — not an email thread. |
| C4 | **No receipt.** | Deposit receipt and payout receipt, both exportable, both referencing the same case. |
| C5 | **No notifications.** Everything requires the customer to refresh a page. | Push/SMS/email on every state change that changes who acts next. |
| C6 | **No admin-on-phone design.** The founder runs operations from a phone, often while meeting the customer. | Operator screens are specified mobile-first, not as a squeezed desktop table. |

## Ordered next fixes

**P0 — launch-blocking.** A1 ownership proof, A2 finality and reorg, A6 double-credit key,
A3/A4/A5 payment-mismatch states, B1/B2/B3 decision record, B5 hash-chained log, B6 degraded mode,
A8 sweep verification, A9 payout acknowledgment, C1/C2 customer pay and track screens, C4 receipts.

**P1 — first 90 days after launch.** A7 unexpected-outflow alerts, B4 four-eyes and solo mode,
B7 SLA clock and ageing, C3 request-for-information flow, C5 notifications, policy editor with
dry-run, dealer-facing API keys and webhooks, multi-chain expansion.

**Polish.** Saved views and batch actions, analytics, print-perfect audit packs, Arabic/RTL and
French localisation, onboarding wizard, dark mode, driver app surface for cash delivery proof.

The rating moves to 9 / 10 when P0 and P1 ship as specified. The last point is reserved: it requires
licensed counsel in the operating jurisdiction to sign off on the payout-hold and no-refund posture,
and a real examiner or auditor to review one exported audit pack and say it is sufficient. Nothing in
this document substitutes for either.
