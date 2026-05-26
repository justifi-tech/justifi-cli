# Fee Configurations

| | |
|--------|----------------------------------------------|
| Status | Ready |
| Last Updated | 2026-05-26 |

## Overview

`justifi fee-configurations` reads and sets the fee rates for a sub-account, per fee type. Maps to the JustiFi **Fee Configurations** tag. The endpoints are nested under a sub-account; a new configuration for a fee type supersedes the prior one (fee history is preserved and queryable).

Inherits all shared command behavior from [resource-command-template.md](resource-command-template.md).

**Sub-account scoping:** every command targets a specific sub-account supplied in the path. The CLI uses the resolved sub-account (`--sub-account`, `JUSTIFI_SUB_ACCOUNT`, or the active profile) as that path id and **requires** one (exit `2` if absent).

## Endpoints

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `list` | GET | `/v1/sub_accounts/{id}/fee_configurations` | `ListFeeConfigurations` |
| `scheduled` | GET | `/v1/sub_accounts/{id}/fee_configurations/scheduled` | `ListScheduledFeeConfigurations` |
| `get` | GET | `/v1/sub_accounts/{id}/fee_configurations/{fee_type}` | `GetFeeConfiguration` |
| `create` | POST | `/v1/sub_accounts/{id}/fee_configurations/{fee_type}` | `CreateFeeConfiguration` |
| `history` | GET | `/v1/sub_accounts/{id}/fee_configurations/{fee_type}/history` | `GetFeeConfigurationHistory` |

## Verbs supported

- `list` — list active fee configurations for the sub-account.
- `scheduled` — list fee configurations scheduled to take effect in the future.
- `get <fee-type>` — fetch the active configuration for one fee type.
- `create <fee-type>` — set a new configuration for a fee type. Supplying a new configuration supersedes the current one for that fee type.
- `history <fee-type>` — list all configurations (past, active, scheduled) for one fee type.

`fee_type` is a positional argument, one of: `processing_ecomm`, `processing_card_present`, `processing_ach`, `processing_ach_expedited`, `visa_brand_ecomm`, `visa_brand_card_present`, `mastercard_brand_ecomm`, `mastercard_brand_card_present`, `amex_brand_ecomm`, `amex_brand_card_present`, `discover_brand_ecomm`, `discover_brand_card_present`, `platform`.

## Resource-specific flags

**`create <fee-type>`:**

| Flag | Required | Maps to | Notes |
|------|----------|---------|-------|
| `--variable-rate` | yes | `variable_rate` | Percent as a number, e.g. `2.75` = 2.75% |
| `--transaction-fee-cents` | no | `transaction_fee_cents` | Default 0 |
| `--fee-cap-cents` | no | `fee_cap_cents` | Cap per transaction |
| `--effective-start` | no | `effective_start` | Defaults to now |
| `--effective-end` | no | `effective_end` | Open-ended when omitted |

## Table columns

**`list` / `scheduled` / `history`:** `id`, `fee_type`, `variable_rate`, `transaction_fee_cents`, `fee_cap_cents`, `effective_start`, `effective_end`.

**`get`:** key/value of the configuration. Notable fields: `id`, `account_id`, `fee_type`, `variable_rate`, `transaction_fee_cents`, `transaction_fee_currency` (`usd`), `fee_cap_cents`, `effective_start`, `effective_end`.

## References

- JustiFi Fee Configurations API: https://docs.justifi.tech/api-spec (Fee Configurations tag)
- [sub-accounts.md](sub-accounts.md)
- [resource-command-template.md](resource-command-template.md)
