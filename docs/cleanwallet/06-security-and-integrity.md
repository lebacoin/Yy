# 06 — Security, money integrity, and compliance posture

CleanWallet holds no keys and moves no money. That removes the largest class of fintech risk and
replaces it with a narrower one: **the product's output is an assurance, and a wrong assurance
causes an irreversible payout.** Every control below exists to protect either the truth of that
assurance or the record of it.

## Threat model

| # | Threat | Impact | Controls |
| --- | --- | --- | --- |
| T1 | **Fake intake address.** Attacker registers an address they do not control, gets a clean assessment, and uses CleanWallet's record to legitimise funds. | Product becomes a laundering instrument | Ownership proof before an address can accept orders (signed message, micro-transfer from a proven address, or descriptor registration); re-proof on any change; `intake_ownership_unproven` blocks order creation and approval |
| T2 | **False clean assessment.** Screening skipped, stale, partial, or provider-spoofed. | Irreversible payout on illicit funds | Degraded mode can never approve; result TTL with forced re-screen; provider responses stored with hash; TLS pinning and response-signature validation where the provider supports it; two-provider agreement above threshold |
| T3 | **Reorg after payout.** Deposit reverses after cash has left. | Direct loss | Per-chain finality thresholds tuned above documented reorg depth; no approval pre-finality; continuous re-verification of matched deposits for a trailing window; `deposit.reversed` voids assessments and raises P1 |
| T4 | **Double credit.** Same transaction pays two orders. | Direct loss | Unique `(chain, tx_hash, output_index, asset)`; one intake address per order; reconciliation job comparing chain totals to credited totals daily |
| T5 | **Evidence tampering.** Someone edits history after an examiner asks. | Total loss of product value | Append-only events with `UPDATE`/`DELETE` revoked at the database role; hash chain computed by trigger; periodic anchoring; exports carry the chain; separate write path for corrections (supersede, never edit) |
| T6 | **Insider approval abuse.** Operator approves illicit funds, or edits a policy to make them pass. | Regulatory and criminal exposure | Reason codes and rationale mandatory; recommendation stored separately from decision so overrides are visible; four-eyes above threshold; policy publish is dual-controlled, versioned, non-retroactive, with a diff and an approver; exception register surfaces every override |
| T7 | **Account takeover of an operator.** | Fraudulent approvals, data exfiltration | Passkeys/WebAuthn mandatory for `owner` and `operator`; step-up auth on decisions, policy publish, address registry changes, and exports; device/session list with revoke; IP allowlist option; anomaly alerts on new device plus first approval |
| T8 | **Dealer compromise moving funds out of intake.** | Funds gone; CleanWallet cannot prevent it | `outflow.unexpected` within one block for any transfer to an unregistered destination; full-bleed operator alert; recorded as evidence — detection is the deliverable, not prevention |
| T9 | **Customer PII and screening-data leakage.** Screening results are sensitive personal data. | Legal exposure, tipping-off | Field-level encryption for documents and identity fields; storage-side encryption with per-dealer keys; role-based redaction (`driver` sees a name and an address, never a case); no PII or outcome in push previews, subject lines, or logs; export of personal data is step-up authenticated and logged |
| T10 | **Tipping off through the customer UI.** Neutral copy leaks the fact of a report or a hit. | Criminal offence in several regimes | One neutral vocabulary enforced in [`03-screens-customer.md`](03-screens-customer.md); a lint rule over customer-facing copy strings blocking a banned-words list; report-to-authority records store only a reference, never content, and never surface customer-side |
| T11 | **Webhook forgery** into a dealer's own systems. | Bogus "approved" upstream | HMAC-SHA256 with timestamp, replay window, rotating secrets, documented verification snippet; consumers required to be idempotent on `event.id` |
| T12 | **Provider dependency and lock-in.** Single KYT vendor down, or terms change. | Operations stop | Provider abstraction with a configured fallback; outages recorded as evidence; contractual check that screening data may be retained in audit packs for the retention period |
| T13 | **Wrong-network funds turned into a support crisis.** | Trust damage, informal key handling | Explicit `unsupported_asset` state with an honest recoverability statement; recovery, where possible, is performed by the dealer with the dealer's keys — CleanWallet never asks for, receives, or stores a key or seed phrase, and the UI says so at the moment of highest temptation |
| T14 | **Sweep to the wrong destination.** | Loss, or funds outside the record | Treasury allowlist; sweep verification on source, destination, asset, amount, finality; `sweep.mismatch` blocks payout authorization |
| T15 | **Payout without proof.** "We paid" with nothing behind it. | Disputes, chargeback-style losses | Payout requires a verified sweep; acknowledgment via handover code hash, customer confirmation, or transfer reference; driver acknowledgment is scoped and timestamped |

## Money-integrity rules

Seven rules that no configuration, role, or feature flag may override:

1. **No approval before finality.**
2. **No approval on a degraded or missing required screening result.**
3. **No approval on an address whose ownership is unproven.**
4. **No payout record without a verified sweep.**
5. **No automatic return, refund, or onward transfer of funds — ever, by anyone, for any reason.**
6. **No decision without a structured reason code and a rationale.**
7. **No edit or deletion of a recorded event; corrections supersede.**

These belong in code as guard clauses with dedicated error codes, in the database as constraints, and
in the test suite as named tests. If a rule can be turned off in a settings screen, it is not a rule.

## Secrets and infrastructure

- No private keys, seed phrases, or signing material in any CleanWallet system, backup, log, or
  support channel. A support macro exists for the case where a customer or dealer offers one: refuse
  and explain.
- Provider credentials and webhook secrets in a managed vault, scoped per worker, rotated on a
  schedule, never in environment variables shared with the public API process.
- Structured logging with an allowlist of loggable fields; addresses truncated; amounts permitted;
  documents, identity fields, and raw screening responses never logged.
- Backups encrypted, restore-tested quarterly, retention matched to the policy's retention period
  (and no longer — over-retention of screening data is its own liability).
- Dependency and container scanning in CI; secret scanning on every push; the public API's egress
  restricted to the provider and RPC allowlists.

## Compliance posture — stated plainly

CleanWallet is **software**. It does not hold client funds, does not transmit value, does not provide
regulated financial services, and does not provide legal advice. The dealer remains the regulated or
obliged party, and the dealer makes every decision. CleanWallet supplies evidence, checks, and a
record.

What this plan deliberately does **not** claim:

- It does not claim to detect illicit funds. It claims to run the checks the dealer configured and to
  record what they returned. **A screening pass is not a guarantee of lawful funds** — on the landing
  page, in the product, in the API response, and in every audit pack.
- It does not claim automated compliance. The automation is detection, verification, screening, and
  recommendation; the decision is human and labelled as such.
- It does not claim regulatory approval, certification, or an audit it has not had.

Obligations that stay with the dealer, and that the UI should name rather than absorb: customer due
diligence and KYC, suspicious-activity reporting, sanctions-screening obligations under their own
licence, record retention periods under local law, and any decision to return funds.

**Before launch, three things require named human sign-off**, and none can be produced by this plan:

1. **Licensed counsel in the operating jurisdiction** confirms the non-custodial posture, the
   no-automatic-return rule, the retention periods, and the customer-facing hold copy against
   tipping-off rules.
2. **The KYT provider's terms** permit retaining and exporting screening results inside audit packs
   for the retention period, and permit the dealer-facing presentation used in O2.
3. **One exported audit pack is reviewed by a real auditor or examiner** who confirms it would be
   sufficient evidence. Until that happens, the pack is a good guess.

Treat all three as launch gates in [`07-build-plan.md`](07-build-plan.md), not as paperwork to chase
afterwards.
