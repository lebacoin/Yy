# CleanWallet — Build Plan

**Check the payment. Protect the deal.**

CleanWallet is the payment-assurance layer for high-value OTC crypto deals. A customer sells
crypto and expects cash (or a transfer). Before the dealer releases that payout, CleanWallet
proves four things and records the proof forever:

1. The payment really arrived, on the right network, in the right amount.
2. It is final on-chain — not reversible.
3. It matches a real order and a real customer.
4. It passed the screening and evidence rules the dealer set.

**CleanWallet is the observer, not the custodian.** No private keys. No shared pool.
No intermediary wallets. No transfer endpoint in the API. The dealer owns the intake address,
the dealer signs every movement of funds, and CleanWallet verifies and records what happened.

## Status

Proposed workflow, not a live product. No backend exists in this repository today — it currently
holds the Lebaneeds Connect / Lebaneeds Payments static marketing site. Everything in this folder
is a specification to build against, not a description of something running.

A screening pass is not a guarantee of lawful funds. That sentence is a product constraint, and it
appears in the UI, in the API docs, and in every exported audit pack.

## How to read this folder

| File | What it decides |
| --- | --- |
| [`01-audit.md`](01-audit.md) | Honest audit of the launch scheme, rating, and the 22 gaps that must close |
| [`02-flow-and-states.md`](02-flow-and-states.md) | The end-to-end flow, every state machine, and the "who acts next" model |
| [`03-screens-customer.md`](03-screens-customer.md) | Customer screens, mobile-first, all states, exact copy |
| [`04-screens-operator.md`](04-screens-operator.md) | Operator/admin screens, including admin-on-phone |
| [`05-data-model-and-api.md`](05-data-model-and-api.md) | Schema, state fields, REST API, webhooks, idempotency |
| [`06-security-and-integrity.md`](06-security-and-integrity.md) | Threats, money/evidence-integrity controls, compliance posture |
| [`07-build-plan.md`](07-build-plan.md) | P0 / P1 / polish, milestones, acceptance criteria |
| [`08-codex-handoff.md`](08-codex-handoff.md) | Ticket-by-ticket implementation spec |

## The one-line architecture

```
Customer wallet ──funds──> Dealer intake address (dealer keys)
                                  │
                                  │ read-only: RPC + indexer + KYT
                                  v
                            CleanWallet ── assessment ──> Operator decision
                                  │                              │
                                  │ verifies                     │ dealer signs
                                  v                              v
                            Sweep receipt <──────── Dealer treasury (dealer keys)
                                  │
                                  v
                     Payout authorization + acknowledgment (cash / driver / transfer)
                                  │
                                  v
                        Append-only, hash-chained evidence record
```

Solid lines are money and only the dealer can create them. Dashed-equivalent lines — reads,
assessments, decisions, records — are all CleanWallet ever produces.
