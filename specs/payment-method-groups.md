# Payment Method Groups

| | |
|--------|----------------------------------------------|
| Status | Ready |
| Last Updated | 2026-05-26 |

## Overview

`justifi payment-method-groups` manages groups of stored payment methods. Maps to the JustiFi **Payment Method Groups** tag. A group is referenced by id (`pmg_…`) and can be attached to payment methods.

Inherits all shared command behavior from [resource-command-template.md](resource-command-template.md).

**Sub-account scoping:** `create` **requires** a resolved sub-account (exit `2` if absent). `list`, `get`, `update`, and `remove-payment-method` accept one to scope the call.

## Endpoints

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `list` | GET | `/v1/payment_method_groups` | `ListPaymentMethodGroup` |
| `get` | GET | `/v1/payment_method_groups/{id}` | `GetPaymentMethodGroup` |
| `create` | POST | `/v1/payment_method_groups` | `CreatePaymentMethodGroup` |
| `update` | PATCH | `/v1/payment_method_groups/{id}` | `PatchPaymentMethodGroup` |
| `remove-payment-method` | DELETE | `/v1/payment_method_groups/{id}/payment_methods/{payment_method_id}` | `RemovePaymentMethodFromGroup` |

## Verbs supported

- `list` — list payment method groups, cursor-paginated.
- `get <id>` — fetch one group.
- `create` — create a group. The endpoint takes no request body; the group is created empty.
- `update <id>` — set the group's payment-method membership.
- `remove-payment-method <id> <payment-method-id>` — remove one payment method from the group.

## Resource-specific flags

**`update <id>`:** `--payment-method-id` (repeatable) → `payment_method_ids[]`. The supplied set replaces the group's membership.

**`remove-payment-method`:** takes two positional arguments — the group id and the payment-method id.

`create` takes no flags beyond the inherited idempotency key. `list` uses the shared pagination flags.

## Table columns

**`list`:** `id`, `account_id`, `platform_account_id`.

**`get`:** key/value of `id`, `account_id`, `platform_account_id`.

## References

- JustiFi Payment Method Groups API: https://docs.justifi.tech/api-spec (Payment Method Groups tag)
- [payment-methods.md](payment-methods.md)
- [resource-command-template.md](resource-command-template.md)
