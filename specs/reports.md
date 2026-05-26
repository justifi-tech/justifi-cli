# Reports

| | |
|--------|----------------------------------------------|
| Status | Ready |
| Last Updated | 2026-05-26 |

## Overview

`justifi reports` generates and retrieves CSV report exports. Maps to the JustiFi **Reports** tag. Report generation is asynchronous: `create` queues a report; `get` returns its status and, once completed, a presigned download URL.

Inherits all shared command behavior from [resource-command-template.md](resource-command-template.md).

**Sub-account scoping:** `create`, `list`, and `get` accept a sub-account to scope the call.

## Endpoints

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `list` | GET | `/v1/reports` | `ListSubAccounts` |
| `get` | GET | `/v1/reports/{id}` | `GetReport` |
| `create` | POST | `/v1/reports` | `CreateReport` |

(The `list` endpoint's upstream operationId is `ListSubAccounts` — a known naming quirk in the OpenAPI spec; the CLI uses it verbatim for codegen but exposes it as `reports list`.)

## Verbs supported

- `list` — list reports, cursor-paginated.
- `get <id>` — fetch one report's status and download URL.
- `create` — queue a new report.

## Resource-specific flags

**`create`:**

| Flag | Required | Maps to | Notes |
|------|----------|---------|-------|
| `--report-type` | yes | `report_type` | `proceeds`, `payout`, `interchange_fee`, `sub_account_summary`, `payment_list` |
| `--start-date` | no | `start_date` | Date; max one-month range |
| `--end-date` | no | `end_date` | Date |
| `--nickname` | no | `nickname` | Label for the report |
| `--payment-status` | no | `payment_status` | `payment_list` only: `authorized`, `failed`, `succeeded`, `canceled` |
| `--payment-method-id` | no | `payment_method_id` | `payment_list` only |
| `--terminal-id` | no | `terminal_id` | `payment_list` only |

**`list` filters:** `--nickname` → `nickname`.

## Table columns

**`list`:** `id`, `report_type`, `nickname`, `status`, `created_at`.

**`get`:** key/value of the full report. Notable fields: `id`, `report_type` (`proceeds`, `payout`, `interchange_fee`, `sub_account_summary`, `payment_list`), `nickname`, `status` (`scheduled`, `processing`, `completed`, `failed`, `canceled`, `expired`), `scheduled_at`, `run_at`, `error_description`, `presigned_url`, `created_at`.

## Special behavior

- **Async generation.** `create` returns a report in `scheduled`/`processing` status. The caller polls `get <id>` until `status` is `completed`, at which point `presigned_url` is populated.
- **Download.** `get <id>` accepts `--download` to fetch the CSV from `presigned_url`; combined with `--output-file <path>` it writes the CSV to a file, otherwise it streams the CSV to stdout. Without `--download`, `get` returns the report metadata per the normal output rules.

## References

- JustiFi Reports API: https://docs.justifi.tech/api-spec (Reports tag)
- [resource-command-template.md](resource-command-template.md)
