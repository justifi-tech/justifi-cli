# Payment Methods

| | |
|--------|----------------------------------------------|
| Status | Ready |
| Last Updated | 2026-05-26 |

## Overview

`justifi payment-methods` tokenizes and manages stored payment methods (cards and bank accounts). Maps to the JustiFi **Payment Methods** tag. Stored payment methods are identified by a token (`pm_…`) and consumed by `payments create`.

Inherits all shared command behavior from [resource-command-template.md](resource-command-template.md).

**Sub-account scoping:** `create` **requires** a resolved sub-account (exit `2` if absent). `list` accepts one to scope results. `get`, `update`, and `clone` operate by token.

## Endpoints

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `list` | GET | `/v1/payment_methods` | `ListPaymentMethods` |
| `get` | GET | `/v1/payment_methods/{token}` | `GetPaymentMethod` |
| `create` | POST | `/v1/payment_methods` | `CreatePaymentMethod` |
| `update` | PATCH | `/v1/payment_methods/{token}` | `UpdatePaymentMethod` |
| `clone` | POST | `/v1/payment_methods/{token}/clone` | `ClonePaymentMethod` |

## Verbs supported

- `list` — list payment methods, cursor-paginated.
- `get <token>` — fetch one payment method by token.
- `create` — tokenize and store a card or bank account.
- `update <token>` — update a stored card's expiration, address, or metadata.
- `clone <token>` — copy a payment method to another account.

## Resource-specific flags

**`create`** — supplies either card details or bank-account details (one group, per the JustiFi `CreateCard` / `CreateBankAccount` schemas):

| Flag | Maps to | Notes |
|------|---------|-------|
| `--card-number`, `--card-exp-month`, `--card-exp-year`, `--card-name`, `--card-cvv`, `--card-address-*` | `payment_method.card` | Card group (per `CreateCard`) |
| `--bank-account-number`, `--bank-routing-number`, `--bank-account-type`, `--bank-account-owner-name` | `payment_method.bank_account` | Bank-account group (per `CreateBankAccount`) |
| `--payment-method-group-id` | `payment_method_group_id` | Add to a group on creation |
| `--email` | `email` | |
| `--force-tokenize` | `force_tokenize` | Tokenize even when a matching method exists |

The card group and bank-account group are mutually exclusive; supplying both is a usage error (exit `2`).

**`update <token>`:** card mutable fields — `--card-exp-month`, `--card-exp-year`, `--card-address-*`.

**`clone <token>`:** `--destination-account-id` (required) → `destination_account_id`.

**`list` filters:** `--payment-method-group-id` → `payment_method_group_id`; `--customer-id` → `customer_id`.

## Table columns

**`list`:** `token`, `type` (card / bank_account), `brand_or_bank`, `last_four`, `status`, `created_at`.

**`get`:** key/value of the full payment method. Card methods show `card.brand`, `card.acct_last_four`, `card.name`, `card.month`, `card.year`, `card.token`; bank-account methods show the bank-account fields. Common fields: `status`, `invalid_reason`, `customer_id`, `account_id`.

## Platform wallet accounts

A platform wallet account is a sub-account with a wallet setting enabled by JustiFi. Payment methods stored on it are managed through these same commands with the wallet account's id supplied as the sub-account: `justifi payment-methods create --sub-account <wallet-id> …`. Use `clone <token>` with `--destination-account-id` to propagate a wallet payment method to another sub-account.

## References

- JustiFi Payment Methods API: https://docs.justifi.tech/api-spec (Payment Methods tag)
- [payment-method-groups.md](payment-method-groups.md)
- [resource-command-template.md](resource-command-template.md)
