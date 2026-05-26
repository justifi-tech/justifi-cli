# Checkouts

| | |
|--------|----------------------------------------------|
| Status | Ready |
| Last Updated | 2026-05-26 |

## Overview

`justifi checkouts` manages hosted and embedded payment sessions: creating a checkout, retrieving and listing checkouts, updating an open checkout, and completing one with a payment token. Maps to the JustiFi **Checkouts** tag.

Inherits all shared command behavior from [resource-command-template.md](resource-command-template.md).

**Sub-account scoping:** `create` **requires** a resolved sub-account (exit `2` if absent). `list` and `get` accept one to scope the call. `complete` operates by checkout id.

## Endpoints

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `list` | GET | `/v1/checkouts` | `ListCheckouts` |
| `get` | GET | `/v1/checkouts/{id}` | `GetCheckout` |
| `create` | POST | `/v1/checkouts` | `CreateCheckout` |
| `update` | PATCH | `/v1/checkouts/{id}` | `UpdateCheckout` |
| `complete` | POST | `/v1/checkouts/{id}/complete` | `CompleteCheckout` |

## Verbs supported

- `list` — list checkouts, cursor-paginated.
- `get <id>` — fetch one checkout.
- `create` — open a checkout session for a given amount.
- `update <id>` — change amount, description, descriptor, or metadata on an open checkout.
- `complete <id>` — complete a checkout against a payment-method token.

## Resource-specific flags

**`create`:**

| Flag | Required | Maps to | Notes |
|------|----------|---------|-------|
| `--amount` | yes | `amount` | Integer, cents |
| `--description` | yes | `description` | |
| `--origin-url` | no | `origin_url` | |
| `--payment-method-group-id` | no | `payment_method_group_id` | |
| `--statement-descriptor` | no | `statement_descriptor` | 5–22 chars |

**`update <id>`:** `--amount`, `--description`, `--statement-descriptor`.

**`complete <id>`:**

| Flag | Required | Maps to | Notes |
|------|----------|---------|-------|
| `--payment-token` | yes | `payment_token` | Payment-method token |
| `--payment-mode` | no | `payment_mode` | `ecom` (default), `bnpl`, `card_present`, `apple_pay` |

**`list` filters:** `--payment-mode` → `payment_mode` (`bnpl`, `ecom`, `card_present`, `apple_pay`); `--status` → `status` (`created`, `completed`, `attempted`, `expired`); `--payment-status` → `payment_status` (`succeeded`, `failed`, `canceled`, `skipped`, `pending`).

## Table columns

**`list`:** `id`, `payment_amount` (rendered with `payment_currency`), `status`, `payment_description`, `successful_payment_id`, `created_at`.

**`get`:** key/value of the full checkout. Notable fields: `id`, `account_id`, `payment_intent_id`, `payment_amount`, `payment_currency` (`USD`, `CAD`), `payment_description`, `status` (`created`, `completed`, `attempted`, `expired`), `mode` (`test`, `live`), `successful_payment_id`, `statement_descriptor`, `created_at`, `updated_at`.

## References

- JustiFi Checkouts API: https://docs.justifi.tech/api-spec (Checkouts tag)
- [resource-command-template.md](resource-command-template.md)
