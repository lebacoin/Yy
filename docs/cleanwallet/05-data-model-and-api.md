# 05 — Data model and API

Conventions follow the existing Lebaneeds Payments API (`docs.html`) so a partner integrates once:
versioned path `/v1/`, bearer keys `sk_test_…` / `sk_live_…`, `Idempotency-Key` on every mutating
request, HMAC-SHA256 signatures on webhook delivery, and the error body
`{ "error": { "type", "code", "message" } }`.

**The API has no transfer endpoint.** There is no way, with any key or any scope, to make CleanWallet
move funds — because CleanWallet holds no keys. This is documented as a feature, not an omission.

## Schema

Postgres. Money as `numeric(38,0)` in the asset's smallest unit plus a separate `decimals` on the
asset — never floats, never a display string.

```sql
-- Tenancy -------------------------------------------------------------------
dealers            (id, legal_name, jurisdiction, status, created_at)
users              (id, dealer_id, email, role, mfa_enrolled_at, status)
                   -- role: owner | operator | reviewer | auditor | driver
api_keys           (id, dealer_id, prefix, hash, scopes[], last_used_at, revoked_at)

-- Dealer-owned addresses (CleanWallet stores no keys, ever) -----------------
addresses          (id, dealer_id, chain, address, purpose, label, status,
                    ownership_proof_method, ownership_proof_ref, ownership_proven_at,
                    derivation_descriptor, created_at)
                   -- purpose: intake | treasury | fee
                   -- proof_method: signed_message | micro_transfer | descriptor
                   UNIQUE (chain, address)

-- Orders --------------------------------------------------------------------
customers          (id, dealer_id, external_ref, display_name, kyc_status, kyc_ref, created_at)
orders             (id, dealer_id, reference, customer_id, asset_id, chain,
                    expected_amount, tolerance_bps, intake_address_id,
                    payout_method, payout_amount, payout_currency,
                    state, awaiting_party, sla_due_at, expires_at, created_by, created_at)
                   -- payout_method: cash_pickup | cash_delivery | bank_transfer | wallet_credit
                   UNIQUE (dealer_id, reference)
                   UNIQUE (intake_address_id)          -- an intake address serves one order, ever

-- On-chain facts ------------------------------------------------------------
assets             (id, chain, symbol, contract_address, decimals, finality_confirmations, status)
deposits           (id, dealer_id, chain, tx_hash, output_index, asset_id, amount,
                    to_address_id, from_addresses jsonb, block_height, first_seen_at,
                    confirmations, state, matched_order_id, reversed_at, created_at)
                   -- state: detected | confirming | final | matched | reversed
                   --        | unmatched | unsupported_asset | ignored
                   UNIQUE (chain, tx_hash, output_index, asset_id)   -- THE money key
outflows           (id, dealer_id, chain, tx_hash, from_address_id, to_address,
                    asset_id, amount, is_registered_destination, alert_state, detected_at)

-- Screening and assessment --------------------------------------------------
screenings         (id, deposit_id, provider, provider_ref, requested_at, received_at,
                    band, score, reasons jsonb, related_addresses jsonb,
                    raw_response jsonb, raw_response_sha256, degraded, ttl_expires_at)
policies           (id, dealer_id, version, document jsonb, effective_from,
                    created_by, approved_by, published_at)
                   UNIQUE (dealer_id, version)
assessments        (id, deposit_id, order_id, policy_id, outcome, recommendation,
                    inputs_sha256, evaluated_at, state, superseded_by, voided_at, voided_reason)
                   -- outcome: satisfied | needs_review | material_risk
                   -- state:   pending | recorded | voided
decisions          (id, assessment_id, action, reason_code, rationale,
                    decided_by, decided_at, second_approver_id, approver_mode)
                   -- action: approve | request_info | decline | close_unresolved
                   -- approver_mode: dual | single_approver

-- Cases, evidence, settlement ----------------------------------------------
cases              (id, dealer_id, order_id, state, outcome, opened_at, closed_at,
                    awaiting_party, sla_due_at, retention_until, report_ref)
rfis               (id, case_id, requested_items jsonb, sent_at, responded_at, state)
documents          (id, case_id, kind, storage_ref, sha256, uploaded_by,
                    uploaded_at, retention_until)
sweeps             (id, order_id, tx_hash, from_address_id, to_address_id, asset_id,
                    amount, fee, state, verified_at, mismatch_reason, registered_by)
payouts            (id, order_id, method, amount, currency, authorized_by, authorized_at,
                    driver_ref, handover_code_hash, state, acknowledged_at, ack_method, ack_ref)

-- Evidence integrity --------------------------------------------------------
events             (id BIGSERIAL, dealer_id, subject_type, subject_id, type, actor_type,
                    actor_id, payload jsonb, payload_sha256, prev_sha256, chain_sha256,
                    occurred_at)
                   -- append-only: no UPDATE, no DELETE grant to the application role
anchors            (id, dealer_id, up_to_event_id, chain_sha256, anchored_at, anchor_ref)
provider_outages   (id, provider, started_at, ended_at, affected_deposit_count)
exports            (id, dealer_id, kind, filters jsonb, requested_by, requested_at, file_sha256)
```

Integrity constraints that must be enforced in the database, not only in code:

1. `deposits` unique money key — the only reliable defence against double credit.
2. `orders.intake_address_id` unique — no address reuse across orders.
3. `events` is append-only: revoke `UPDATE`/`DELETE` from the application role; the hash chain
   (`chain_sha256 = sha256(prev_sha256 || payload_sha256)`) is computed by a trigger, not the app.
4. A partial unique index preventing more than one non-voided `assessment` per `(deposit_id)`.
5. A check constraint: a `payouts` row cannot exist unless its order has a `verified` sweep.
6. Row-level security scoping every table by `dealer_id`; `auditor` role granted `SELECT` only;
   `driver` role scoped to `payouts` rows assigned to them and to no customer document.

## Endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/v1/addresses` | Register a dealer-owned address (intake / treasury / fee) |
| `POST` | `/v1/addresses/{id}/ownership-proof` | Submit signed message, micro-transfer ref, or descriptor |
| `GET` | `/v1/addresses` | List with proof status |
| `POST` | `/v1/orders` | Create an order; allocates a proven, unused intake address |
| `GET` | `/v1/orders/{id}` | Full order with current state and `awaiting_party` |
| `POST` | `/v1/orders/{id}/cancel` | Cancel before payment |
| `POST` | `/v1/orders/{id}/requote` | New rate on an expired or late-paid order |
| `GET` | `/v1/deposits/{id}` | On-chain facts, confirmations, finality |
| `POST` | `/v1/deposits/{id}/match` | Manually match an `unmatched` deposit to an order |
| `POST` | `/v1/deposits/{id}/rescreen` | Force a fresh screening (rate-limited, logged) |
| `GET` | `/v1/assessments/{id}` | Outcome, recommendation, policy version, inputs hash |
| `POST` | `/v1/assessments/{id}/decision` | Record a decision (reason code + rationale required) |
| `POST` | `/v1/cases/{id}/rfi` | Open a request for information |
| `POST` | `/v1/cases/{id}/close` | Close with outcome, incl. `unresolved` |
| `POST` | `/v1/sweeps` | Register a dealer-signed sweep tx hash for verification |
| `GET` | `/v1/sweeps/{id}` | Verification result or mismatch reason |
| `POST` | `/v1/payouts` | Record payout authorization (requires a verified sweep) |
| `POST` | `/v1/payouts/{id}/acknowledge` | Record acknowledgment (handover code, signature, or transfer ref) |
| `GET` | `/v1/audit-packs/{case_id}` | Generate/download the evidence pack |
| `GET` | `/v1/policies` · `POST` `/v1/policies` · `POST` `/v1/policies/{id}/publish` | Versioned policy management |
| `POST` | `/v1/webhooks` | Register an endpoint |

Notably absent, by design: anything that signs, sweeps, transfers, refunds, or returns funds.

### Example — create an order

```http
POST /v1/orders
Authorization: Bearer sk_live_…
Idempotency-Key: order-8412-v1

{
  "reference": "CW-98234",
  "customer": { "external_ref": "cust_5521" },
  "chain": "tron",
  "asset": "USDT",
  "expected_amount": "10000000000",
  "payout": { "method": "cash_delivery", "amount": "990000", "currency": "USD" },
  "expires_in": 1800
}
```

```json
{
  "id": "ord_2f9a…",
  "reference": "CW-98234",
  "state": "awaiting_payment",
  "awaiting_party": "customer",
  "intake_address": { "chain": "tron", "address": "TQn9Y2kh…", "ownership_proven_at": "2026-03-04T09:11:00Z" },
  "expected_amount": "10000000000",
  "asset": { "symbol": "USDT", "decimals": 6, "contract_address": "TR7NHq…", "finality_confirmations": 20 },
  "expires_at": "2026-03-12T14:32:00Z",
  "disclaimer": "A screening pass is not a guarantee of lawful funds."
}
```

An order cannot be created against an address whose `ownership_proven_at` is null — the API returns
`409` with code `intake_ownership_unproven`.

### Webhook events

| Event | Fired when |
| --- | --- |
| `deposit.detected` | First seen on-chain, 0 confirmations |
| `deposit.confirming` | Confirmation count changed (throttled) |
| `deposit.final` | Finality threshold reached |
| `deposit.reversed` | Reorg removed a previously reported deposit |
| `deposit.unsupported_asset` | Wrong network or unknown token received |
| `deposit.mismatch` | Under/over/late/multiple payment |
| `assessment.ready` | Outcome computed and awaiting a decision |
| `assessment.voided` | Deposit reversed or policy recalled |
| `decision.recorded` | Operator decision stored |
| `case.opened` · `case.closed` | Review lifecycle |
| `sweep.verified` · `sweep.mismatch` | Dealer-signed sweep checked |
| `outflow.unexpected` | Funds left an intake address to an unregistered destination |
| `payout.authorized` · `payout.acknowledged` | Settlement legs recorded |
| `screening.degraded` · `screening.restored` | Provider outage boundaries |

Delivery: signed `CleanWallet-Signature: t=…,v1=…` (HMAC-SHA256 over `t.body`), exponential backoff
for 24 hours, monotonic `sequence` per order so out-of-order delivery is detectable, and an
at-least-once contract — consumers must be idempotent on `event.id`.

### Errors specific to CleanWallet

| Code | Status | Meaning |
| --- | --- | --- |
| `intake_ownership_unproven` | 409 | Address cannot accept orders until proof is recorded |
| `address_already_assigned` | 409 | Intake addresses are never reused |
| `deposit_not_final` | 409 | Approval attempted before finality |
| `screening_unavailable` | 409 | Degraded mode — approval is not permitted |
| `duplicate_deposit` | 409 | Money key already recorded |
| `reason_code_required` | 400 | Decisions require a structured reason |
| `second_approver_required` | 409 | Above the four-eyes threshold |
| `sweep_destination_unregistered` | 422 | Sweep target is not a registered treasury address |
| `no_verified_sweep` | 409 | Payout recorded before sweep verification |
| `transfer_not_supported` | 501 | CleanWallet does not move funds — returned on any transfer attempt |

## Services

Four processes, deliberately separated:

1. **Public API** — stateless, no chain credentials, no provider secrets beyond its own vault scope.
2. **Chain watcher** — one worker per chain; RPC + indexer with two independent sources and a
   disagreement alarm; tracks confirmations, emits reversals, never writes decisions. No outbound
   internet beyond its RPC allowlist.
3. **Screening worker** — holds provider credentials, retries, snapshots raw responses, records
   outages. Isolated so a provider-side compromise cannot read the order book.
4. **Outbox dispatcher** — reads committed events, delivers webhooks and notifications. Nothing else
   is allowed to send outbound messages, which makes "what did we tell the customer" answerable.

Decision path idempotency: `Idempotency-Key` on the decision endpoint plus the partial unique index
on non-voided assessments means a double-tapped `Approve` cannot produce two decisions.
