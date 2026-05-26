# Terminals

| | |
|--------|----------------------------------------------|
| Status | Ready |
| Last Updated | 2026-05-26 |

## Overview

`justifi terminals` manages card-present terminal devices: listing and retrieving terminals, updating a terminal's nickname, checking live device status, and identifying a device. Maps to the JustiFi **Terminals** tag. Terminals are provisioned through terminal orders (see [terminal-orders.md](terminal-orders.md)).

Inherits all shared command behavior from [resource-command-template.md](resource-command-template.md).

**Sub-account scoping:** `list` accepts a sub-account to scope results. The remaining verbs operate by terminal id.

## Endpoints

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `list` | GET | `/v1/terminals` | `listTerminals` |
| `get` | GET | `/v1/terminals/{id}` | `getTerminal` |
| `update` | PATCH | `/v1/terminals/{id}` | `updateTerminal` |
| `status` | GET | `/v1/terminals/{id}/status` | `getTerminalStatus` |
| `identify` | POST | `/v1/terminals/{id}/identify` | `postIdentifyTerminal` |

## Verbs supported

- `list` — list terminals, cursor-paginated.
- `get <id>` — fetch one terminal.
- `update <id>` — update a terminal's nickname.
- `status <id>` — fetch the terminal's live connection status.
- `identify <id>` — flash the terminal's nickname and serial on the physical device for ~20 seconds.

## Resource-specific flags

**`update <id>`:** `--nickname` → `nickname`.

**`list` filters:** `--status` → `status` (`connected`, `disconnected`, `unknown`, `pending_configuration`, `archived`; comma-separated); `--terminal-id` → `terminal_id`; `--provider-id` → `provider_id`; `--terminal-order-id` → `terminal_order_id`; `--verified-after` → `verified_after`; `--verified-before` → `verified_before`; `--verified-on` → `verified_on`.

## Table columns

**`list`:** `id`, `nickname`, `status`, `model_name`, `provider_serial_number`, `verified_at`.

**`get`:** key/value of the full terminal. Notable fields: `id`, `account_id`, `provider` (`verifone`, `verifone_simulator`), `status` (`connected`, `disconnected`, `unknown`, `pending_configuration`, `archived`), `provider_id`, `provider_serial_number`, `nickname`, `model_name`, `verified_at`, `created_at`, `updated_at`.

## Special behavior

- **`identify`** returns no body; on success the CLI reports `identify <id> ok` (table) or `{"data": null, "action": "identify", "resource_id": "<id>"}` (JSON).

## References

- JustiFi Terminals API: https://docs.justifi.tech/api-spec (Terminals tag)
- [terminal-orders.md](terminal-orders.md)
- [resource-command-template.md](resource-command-template.md)
