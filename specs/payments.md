# Payments

| | |
|--------|----------------------------------------------|
| Status | Ready |
| Last Updated | 2026-05-26 |

## Overview

`justifi payments` manages JustiFi payments — charging a stored payment method, retrieving and listing payments, capturing authorized payments, voiding them, and issuing refunds against them. Maps to the JustiFi **Payments** tag.

This spec inherits all shared command behavior from [resource-command-template.md](resource-command-template.md): global flags, output shape, error/exit-code handling, idempotency, and pagination. Only payments-specific detail is below.

**Sub-account scoping:** `create` **requires** a resolved sub-account (`--sub-account`, `JUSTIFI_SUB_ACCOUNT`, or the active profile); the CLI errors with exit `2` if none is resolved. `list` accepts a sub-account to scope results but does not require one. `get`, `update`, `capture`, `void`, and `refund` do not use it.

## Endpoints

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `list` | GET | `/v1/payments` | `ListPayments` |
| `get` | GET | `/v1/payments/{id}` | `GetPayment` |
| `create` | POST | `/v1/payments` | `CreatePayment` |
| `update` | PATCH | `/v1/payments/{id}` | `UpdatePayment` |
| `capture` | POST | `/v1/payments/{id}/capture` | `CapturePayment` |
| `void` | POST | `/v1/payments/{id}/void` | `VoidPayment` |
| `refund` | POST | `/v1/payments/{id}/refunds` | `CreateRefund` |
| `balance-transactions` | GET | `/v1/payments/{id}/payment_balance_transactions` | `GetPaymentBalanceTransactions` |

## Verbs supported

- `list` — list payments for the active sub-account, cursor-paginated.
- `get <id>` — fetch one payment.
- `create` — authorize and (optionally) capture a payment against a stored payment-method token.
- `update <id>` — update mutable fields (`description`, `metadata`) on a payment.
- `capture <id>` — capture a previously authorized (`capture_strategy=manual`) payment. Captures the full authorized amount; the JustiFi endpoint takes no partial-capture amount.
- `void <id>` — cancel an authorized, uncaptured payment. Resulting status is `canceled`.
- `refund <id>` — issue a refund against a payment (full or partial). Refund creation lives here because the endpoint is nested under payments; refund listing and retrieval live in [refunds.md](refunds.md).
- `balance-transactions <id>` — list the ledger entries (fees, settlement) tied to a payment.

## Resource-specific flags

**`create`:**

| Flag | Required | Maps to | Notes |
|------|----------|---------|-------|
| `--amount` | yes | `amount` | Integer, smallest currency unit (cents) |
| `--currency` | yes | `currency` | `usd` or `cad` |
| `--capture-strategy` | yes | `capture_strategy` | `automatic` (auth+capture) or `manual` (auth-only) |
| `--payment-method-token` | yes | `payment_method.token` | Stored payment-method token (`pm_…`) |
| `--email` | no | `email` | |
| `--description` | no | `description` | |
| `--statement-descriptor` | no | `statement_descriptor` | 5–22 chars |
| `--application-fee-amount` | no | `application_fee_amount` | Platforms only; mutually exclusive with `--fee` |
| `--fee` | no (repeatable) | `fees[]` | `type=amount` form, e.g. `--fee processing_fee=120`; mutually exclusive with `--application-fee-amount` |
| `--expedited` | no | `expedited` | ACH only |

Payments are created from a stored payment-method token.

**`update`:** `--description`.

**`refund`:**

| Flag | Required | Maps to | Notes |
|------|----------|---------|-------|
| `--amount` | no | `amount` | Omit for full refund; must be ≤ refundable amount |
| `--reason` | no | `reason` | `duplicate`, `fraudulent`, or `customer_request` |
| `--description` | no | `description` | |
| `--fee` | no (repeatable) | `fees[]` | `type=amount` form |

**`capture` / `void`:** no resource-specific flags.

**`list` filters** (in addition to template's `--limit`/`--after`/`--before`):

| Flag | Maps to | Notes |
|------|---------|-------|
| `--status` | `payment_status` | `pending`, `authorized`, `canceled`, `succeeded`, `failed`, `partially_refunded`, `fully_refunded`, `disputed` |
| `--payment-method-id` | `payment_method_id` | |
| `--void-id` | `void_id` | |
| `--created-after` | `created_after` | RFC 3339 or relative (template convention) |
| `--created-before` | `created_before` | |

## Table columns

**`list`:** `id`, `amount` (rendered `10.00 USD`), `status`, `captured`, `payment_mode`, `description`, `created_at`.

**`get`:** key/value of the full payment object. Default key order per template (`id` first, `created_at`/`updated_at` last). Notable fields: `id`, `amount`, `amount_refundable`, `amount_refunded`, `amount_disputed`, `fee_amount`, `currency`, `status`, `capture_strategy`, `captured`, `payment_mode`, `description`, `metadata`, `is_test`, `error_code`, `created_at`, `updated_at`.

## Special behavior

- **`capture` settles the full authorized amount.** Partial settlement is achieved by capturing, then refunding the difference.
- **`refund` creation lives here.** [refunds.md](refunds.md) exposes `list` and `get` for refunds; creating a refund is done against its payment via `justifi payments refund <id>`.

## References

- JustiFi Payments API: https://docs.justifi.tech/api-spec (Payments tag)
- [resource-command-template.md](resource-command-template.md) — inherited command behavior
- [refunds.md](refunds.md) — refund listing/retrieval
- [balance-transactions.md](balance-transactions.md) — account-level ledger (payment-level lives here as `balance-transactions <id>`)
