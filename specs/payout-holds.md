# Payout Holds

| | |
|--------|----------------------------------------------|
| Status | Ready |
| Last Updated | 2026-05-26 |

## Overview

`justifi payout-holds` lists and retrieves holds placed on payouts for risk or compliance reasons. Maps to the JustiFi **Payout Holds** tag. This is a read-only resource.

Inherits all shared command behavior from [resource-command-template.md](resource-command-template.md).

**Sub-account scoping:** payout-hold endpoints are scoped by credentials; they do not take a sub-account header.

## Endpoints

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `list` | GET | `/v1/payout_holds` | `ListPayoutHolds` |
| `get` | GET | `/v1/payout_holds/{payout_hold_id}` | `GetPayoutHold` |

## Verbs supported

- `list` — list payout holds, cursor-paginated.
- `get <id>` — fetch one payout hold.

## Resource-specific flags

**`list` filters:** `--created-after` → `created_after`; `--created-before` → `created_before`; `--start-date-after` → `start_date_after`; `--start-date-before` → `start_date_before`; `--end-date-after` → `end_date_after`; `--end-date-before` → `end_date_before`.

## Table columns

**`list`:** `id`, `hold_type`, `issued_by`, `active`, `start_date`, `created_at`.

**`get`:** key/value of the full payout hold. Notable fields: `id`, `account_id`, `hold_type` (`first_payment`, `manual`), `issued_by` (`system`, `platform`), `start_date`, `end_date`, `active`, `created_at`.

## References

- JustiFi Payout Holds API: https://docs.justifi.tech/api-spec (Payout Holds tag)
- [resource-command-template.md](resource-command-template.md)
