# Refunds

| | |
|--------|----------------------------------------------|
| Status | Ready |
| Last Updated | 2026-05-26 |

## Overview

`justifi refunds` lists and retrieves refunds and updates their metadata. Maps to the JustiFi **Refunds** tag. Refund creation is performed against a payment via `justifi payments refund <id>` (see [payments.md](payments.md)), since the create endpoint is nested under the payment.

Inherits all shared command behavior from [resource-command-template.md](resource-command-template.md).

**Sub-account scoping:** `list` accepts a sub-account to scope results. `get` and `update` operate by refund id.

## Endpoints

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `list` | GET | `/v1/refunds` | `ListRefunds` |
| `get` | GET | `/v1/refunds/{id}` | `GetRefund` |
| `update` | PATCH | `/v1/refunds/{id}` | `UpdateRefund` |

## Verbs supported

- `list` — list refunds, cursor-paginated.
- `get <id>` — fetch one refund.
- `update <id>` — replace a refund's metadata.

## Resource-specific flags

`update` carries no resource-specific flags; it updates metadata via the inherited `--metadata` flag. `list` carries no resource-specific filters.

## Table columns

**`list`:** `id`, `payment_id`, `amount` (rendered `10.00 USD`), `status`, `reason`, `created_at`.

**`get`:** key/value of the full refund. Notable fields: `id`, `payment_id`, `amount`, `description`, `reason` (`duplicate`, `fraudulent`, `customer_request`), `status` (`pending`, `succeeded`, `failed`), `returned_fees`, `metadata`, `created_at`, `updated_at`.

## References

- JustiFi Refunds API: https://docs.justifi.tech/api-spec (Refunds tag)
- [payments.md](payments.md) — refund creation (`payments refund <id>`)
- [resource-command-template.md](resource-command-template.md)
