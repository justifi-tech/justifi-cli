# Balance Transactions

| | |
|--------|----------------------------------------------|
| Status | Ready |
| Last Updated | 2026-05-26 |

## Overview

`justifi balance-transactions` lists and retrieves ledger entries — the individual debits and credits (payments, fees, refunds, payouts) that make up an account balance. Maps to the JustiFi **Balance Transactions** tag. This is a read-only resource.

Inherits all shared command behavior from [resource-command-template.md](resource-command-template.md).

**Sub-account scoping:** `list` accepts a sub-account to scope results. `get` operates by id. A list call is scoped to a single sub-account or a single payout.

## Endpoints

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `list` | GET | `/v1/balance_transactions` | `ListBalanceTransactions` |
| `get` | GET | `/v1/balance_transactions/{id}` | `GetBalanceTransaction` |

## Verbs supported

- `list` — list balance transactions, cursor-paginated.
- `get <id>` — fetch one balance transaction.

## Resource-specific flags

**`list` filters:** `--payout-id` → `payout_id`; `--source-payment-id` → `source_payment_id`.

## Table columns

**`list`:** `id`, `amount` (rendered `10.00 USD`), `net`, `fee`, `txn_type`, `created_at`.

**`get`:** key/value of the full balance transaction. Notable fields: `id`, `amount`, `net`, `fee`, `currency`, `txn_type`, `source_type`, `payout_id`, `available_on`, `created_at`.

## References

- JustiFi Balance Transactions API: https://docs.justifi.tech/api-spec (Balance Transactions tag)
- [payments.md](payments.md) — payment-scoped ledger via `payments balance-transactions <id>`
- [resource-command-template.md](resource-command-template.md)
