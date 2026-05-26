# Sub Accounts

| | |
|--------|----------------------------------------------|
| Status | Ready |
| Last Updated | 2026-05-26 |

## Overview

`justifi sub-accounts` lists and retrieves platform sub-accounts (merchant accounts) and reads their settings and payout account. Maps to the JustiFi **Sub Accounts** tag. New sub-accounts are created through the entities onboarding flow (see [entities.md](entities.md)); the legacy create endpoint remains available.

Inherits all shared command behavior from [resource-command-template.md](resource-command-template.md).

**Sub-account scoping:** these endpoints identify the sub-account by path; they do not take a sub-account header.

## Endpoints

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `list` | GET | `/v1/sub_accounts` | `ListSubAccounts` |
| `get` | GET | `/v1/sub_accounts/{id}` | `GetSubAccount` |
| `create` | POST | `/v1/sub_accounts` | `CreateSubAccount` |
| `settings` | GET | `/v1/sub_accounts/{id}/settings` | `GetSubAccountSettings` |
| `payout-account` | GET | `/v1/sub_accounts/{id}/payout_account` | `GetPayoutAccount` |

## Verbs supported

- `list` — list sub-accounts, cursor-paginated.
- `get <id>` — fetch one sub-account.
- `create` — create a sub-account by name (legacy; the entities flow is preferred for new platforms).
- `settings <id>` — fetch a sub-account's settings.
- `payout-account <id>` — fetch a sub-account's payout account.

## Resource-specific flags

**`create`:** `--name` (required) → `name`.

**`list` filters:** `--status` → `status` (`created`, `submitted`, `information_needed`, `rejected`, `enabled`, `disabled`, `archived`); `--business-id` → `business_id`.

## Table columns

**`list`:** `id`, `name`, `status`, `account_type`, `currency`, `created_at`.

**`get`:** key/value of the full sub-account. Notable fields: `id`, `name`, `account_type` (`live`, `test`), `status` (`created`, `submitted`, `information_needed`, `rejected`, `enabled`, `disabled`, `archived`), `currency` (`usd`, `cad`), `platform_account_id`, `payout_account_id`, `business_id`, `processing_ready`, `payout_ready`, `payments_activated_on`, `created_at`, `updated_at`.

## References

- JustiFi Sub Accounts API: https://docs.justifi.tech/api-spec (Sub Accounts tag)
- [entities.md](entities.md) — the onboarding flow that provisions new sub-accounts
- [fee-configurations.md](fee-configurations.md) — fee rates per sub-account
- [resource-command-template.md](resource-command-template.md)
