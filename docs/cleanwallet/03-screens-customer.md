# 03 — Customer screens

Mobile-first, 375 px baseline, responsive to tablet and desktop. White background, light-blue
accents, navy ink, clean cards, one main CTA per screen. No fintech jargon: the words "KYT",
"sweep", "risk score", "AML", "sanctions", "quarantine" and "blocked" never appear on a customer
screen. Every screen states who acts next.

## Copy rules (non-negotiable)

1. **Never reveal a screening outcome.** Not the band, not the reason, not the provider. This is a
   tipping-off control, not a style preference.
2. **Never say "flagged", "suspicious", "investigation", "blocked".** The neutral vocabulary is:
   "checking", "verification in progress", "additional information needed", "our team is reviewing".
3. **Never promise a time you do not control.** "Usually within 10 minutes" is allowed when measured.
   "Almost done" is not.
4. **Always give the customer one next action, or tell them plainly there is nothing to do.**
5. **Numbers are exact and repeated back.** Amount, asset, network, and address are shown in full.
6. **Say "your funds are safe" only where it is true** — funds at a dealer intake address held under
   an open case are safe from loss, and the customer should be told so; funds on a wrong network may
   not be recoverable, and that must be said instead.

## C1 — Order / Pay

The screen the customer lands on after agreeing a deal.

```
┌─────────────────────────────────┐
│  CleanWallet            Help ?  │
├─────────────────────────────────┤
│                                 │
│  Send 10,000 USDT               │   H1, navy, 28/34
│  Tron network (TRC-20)          │   accent chip, always visible
│                                 │
│  ┌───────────────────────────┐  │
│  │      [ QR code ]          │  │   card, radius 16, shadow-sm
│  │                           │  │
│  │  TQn9Y2khEsLJW1ChVWFMSMeR │  │   mono, 2 lines, full address
│  │  DFvBt9qXvY                │  │
│  │                           │  │
│  │  [ Copy address ]         │  │   secondary button, full width
│  └───────────────────────────┘  │
│                                 │
│  Order CW-98234                 │
│  Expires in 29:41               │   live countdown, amber under 5:00
│                                 │
│  You receive                    │
│  $9,900 cash — driver delivery  │
│                                 │
│  ⓘ Send only USDT on Tron to    │   info card, light blue wash
│    this address. Another        │
│    network may not be           │
│    recoverable.                 │
│                                 │
│  ┌───────────────────────────┐  │
│  │   I've sent the payment   │  │   PRIMARY CTA, full width, 52 px
│  └───────────────────────────┘  │
│                                 │
│  Payment not detected yet.      │
│  We check the network           │
│  automatically — you don't      │
│  need the transaction hash.     │
│                                 │
│  Add transaction hash (optional)│   text link, opens sheet
└─────────────────────────────────┘
```

- The address is per-order and never reused. If the screen is reloaded, the same address appears.
- `I've sent the payment` does **not** advance the order state. It switches the screen to the tracker
  and starts polling at a higher frequency. Detection is always on-chain. This is the honest version
  of "automatic": the customer's tap is a hint, not a source of truth.
- The optional hash sheet accepts a hash to speed up matching and says exactly that: "This only helps
  us find your payment faster. We'll find it either way."
- Expiry is a quote expiry, not a funds deadline. Copy under the countdown: "After this, we'll
  re-quote at the current rate. Funds sent late are never lost."

**States:** loading (skeleton card, address masked), address-allocation-failed (see error pattern),
expired (countdown replaced with `Order expired` + primary CTA `Get a new quote`), cancelled
(read-only card, `Start a new order`), offline (banner: "You're offline. The address above is still
correct.").

## C2 — Payment tracker

One screen, six honest states. The stepper is always four steps — Sent, Confirmed, Checked, Paid —
so the customer can see the whole journey from the first second.

| State | Headline | Body | CTA | Who acts next |
| --- | --- | --- | --- | --- |
| Waiting | "Waiting for your payment" | "We're watching the network. This screen updates by itself." | — | Customer |
| Detected | "Payment found" | "3 of 20 network confirmations. Usually about 3 minutes." | — | Blockchain |
| Confirmed | "Payment confirmed" | "Running our standard checks on this payment." | — | Us |
| Info needed | "One thing needed" | "To complete this payment we need a document from you." | `Continue` | Customer |
| Reviewing | "Our team is reviewing this payment" | "This is a manual check. We'll update you here and by SMS. Your funds are safe." | `Contact support` | Us |
| Paid | "Payment complete" | "$9,900 delivered and confirmed at 14:32." | `View receipt` | Nobody |

Plus the mismatch states from the matrix in [`02-flow-and-states.md`](02-flow-and-states.md), which
render as an amber card above the stepper with the exact numbers and a single CTA.

Progressive honesty on timing: under 10 minutes show live progress; past the expected window, replace
the estimate with "This is taking longer than usual. Nothing is wrong with your funds — we'll update
you within the hour." Never let a spinner run without changing its message.

**States:** loading (stepper skeleton), error (see pattern), reversed deposit — neutral copy only:
"The network reversed this transaction. Your funds were returned to your wallet by the network. We
can re-quote whenever you're ready." — never implies customer fault.

## C3 — Information request

Only appears when policy requires evidence. Checklist, not a form wall.

```
One thing needed                     H1
To complete this payment, please add:

☐  Proof of source of funds          card, tap to open
   Exchange statement, sale
   agreement, or payslip.
   PDF or photo.

☑  Photo ID                          completed, green check
   Added 12 Mar

Why we ask                           collapsible, closed by default
We're required to keep a record of
where large payments come from. We
ask every customer the same
questions. Your documents are
encrypted and only used for this
check.

[ Submit ]                           PRIMARY, disabled until required items done
```

- Never state or imply the request was triggered by a screening result.
- Upload: camera capture on mobile, 10 MB per file, PDF/JPG/PNG/HEIC, client-side size feedback,
  retry on failure, and a visible list of what was already sent so nothing is asked twice.
- After submit: "Received. Our team will review this — usually within a few hours. We'll text you."
  `awaiting_party` flips to `dealer` and the tracker reflects it.

## C4 — Payout tracking (cash / driver)

| State | Headline | Detail |
| --- | --- | --- |
| Authorized | "Payout approved" | "Preparing your $9,900 for delivery." |
| Dispatched | "On the way" | Driver first name, masked phone, ETA window, `Call driver` |
| Arrived | "Driver has arrived" | Handover code, shown large, mono |
| Acknowledged | "Payment complete" | Amount, time, `View receipt` |
| Failed | "Delivery didn't complete" | Plain reason + `Contact support`. Funds status stated explicitly. |

The handover code is the acknowledgment proof: the customer reads it to the driver, the driver enters
it, and that event is what closes the payout. Cash pickup and bank transfer variants replace the
driver block with a branch address + hours, or a transfer reference + expected arrival.

## C5 — Receipt

One card, exportable to PDF, shareable. Contains: order reference, date/time, asset, network, amount
received, transaction hash (linked to a block explorer), payout method, payout amount,
acknowledgment time, dealer legal name and contact. It does **not** contain screening results, risk
bands, reviewer names, or internal notes — the customer receipt and the audit pack are different
documents with different audiences.

## C6 — Support / dispute

Entry from every screen. Pre-fills the order reference so the customer never has to find it. Shows
the current state and who acts next in the same words as the tracker, so support and UI never
contradict each other. One CTA: `Send message`. Response-time commitment shown only if real.

## Shared patterns

**Error pattern.** Title states what failed in plain words, body states what is true about the
money, CTA is a real recovery action, and a reference code is always shown. Example: "We couldn't
load this order. Your payment is not affected. Try again, or contact support with code CW-98234-L3."

**Empty pattern.** Never a bare "No data". Always: what would appear here, and the one action that
creates it.

**Loading pattern.** Skeletons that match the final layout, never a full-screen spinner over content
that was already correct. Money figures never render until confirmed by the server — a wrong number
that corrects itself destroys more trust than a half-second delay.

**Accessibility.** WCAG AA contrast on the light-blue accents (the accent is for surfaces and
borders; text on white uses navy). Minimum 44 px touch targets. Status conveyed by icon plus text,
never colour alone. Live regions announce state changes. Countdown has a text alternative.
`prefers-reduced-motion` disables the stepper animation.

**Responsive.** 375 px single column; 768 px two columns with the tracker pinned right; 1024 px+
centred 720 px content column — never a full-width stretched form.
