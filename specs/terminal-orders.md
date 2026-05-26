# Terminal Orders

| | |
|--------|----------------------------------------------|
| Status | Ready |
| Last Updated | 2026-05-26 |

## Overview

`justifi terminal-orders` lists, retrieves, and creates orders for physical card-present terminals. Maps to the JustiFi **Terminal Orders** tag. A completed order provisions terminals that then appear in [terminals.md](terminals.md).

Inherits all shared command behavior from [resource-command-template.md](resource-command-template.md).

**Sub-account scoping:** these endpoints do not take a sub-account header; the sub-account is supplied in the request body or query.

## Endpoints

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `list` | GET | `/v1/terminals/orders` | `ListTerminalsOrders` |
| `get` | GET | `/v1/terminals/orders/{id}` | `GetTerminalsOrder` |
| `create` | POST | `/v1/terminals/orders` | `terminalsOrder` |

## Verbs supported

- `list` — list terminal orders, cursor-paginated.
- `get <id>` — fetch one terminal order.
- `create` — order one or more terminals.

## Resource-specific flags

**`create`:**

| Flag | Required | Maps to | Notes |
|------|----------|---------|-------|
| `--business-id` | yes | `business_id` | |
| `--sub-account-id` | yes | `sub_account_id` | Body field (`acc_…`), distinct from the global `--sub-account` header |
| `--order-type` | yes | `order_type` | `boarding_only` or `boarding_shipping` |
| `--order-item` | yes (repeatable) | `order_items[]` | `model=quantity` form, e.g. `--order-item V400m=2`; models `V400m`, `P400`, `E285` |

**`list` filters:** `--created-after` → `created_after`; `--created-before` → `created_before`; `--order-type` → `order_type` (`boarding_only`, `boarding_shipping`); `--order-status` → `order_status` (`created`, `submitted`, `completed`); `--sub-account-id` → `sub_account_id`.

## Table columns

**`list`:** `id`, `order_type`, `order_status`, `company_name`, `created_at`.

**`get`:** key/value of the full terminal order. Notable fields: `id`, `business_id`, `account_id`, `order_type`, `order_status` (`created`, `submitted`, `in_progress`, `completed`, `on_hold`, `canceled`), `company_name`, `mcc`, `receiver_name`, address fields, `shipping_tracking_reference`, `terminals[]` (`terminal_id`, `terminal_did`, `model_name`), `created_at`, `updated_at`.

## References

- JustiFi Terminal Orders API: https://docs.justifi.tech/api-spec (Terminal Orders tag)
- [terminals.md](terminals.md)
- [resource-command-template.md](resource-command-template.md)
