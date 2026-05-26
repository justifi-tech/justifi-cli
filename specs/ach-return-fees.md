# ACH Return Fees

| | |
|--------|----------------------------------------------|
| Status | Ready |
| Last Updated | 2026-05-26 |

## Overview

`justifi ach-return-fees` retrieves a fee charged when an ACH payment is returned. Maps to the JustiFi **ACH Return Fees** tag. The API exposes a single fetch-by-id endpoint.

Inherits all shared command behavior from [resource-command-template.md](resource-command-template.md).

**Sub-account scoping:** the endpoint is scoped by credentials; it does not take a sub-account header.

## Endpoints

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `get` | GET | `/v1/ach_return_fees/{id}` | `GetAchReturnFee` |

## Verbs supported

- `get <id>` — fetch one ACH return fee (`arf_…`).

## Resource-specific flags

None beyond the inherited global flags.

## Table columns

**`get`:** key/value of the ACH return fee. Fields: `id`, `payment_id`, `amount` (rendered `1.50 USD`), `currency` (`usd`), `created_at`, `updated_at`.

## References

- JustiFi ACH Return Fees API: https://docs.justifi.tech/api-spec (ACH Return Fees tag)
- [resource-command-template.md](resource-command-template.md)
