# Proceeds

| | |
|--------|----------------------------------------------|
| Status | Ready |
| Last Updated | 2026-05-26 |

## Overview

`justifi proceeds` lists and retrieves proceeds — the platform's share of payment fees, settled on its own schedule. Maps to the JustiFi **Proceeds** tag. This is a read-only resource.

Inherits all shared command behavior from [resource-command-template.md](resource-command-template.md).

**Sub-account scoping:** proceeds endpoints are scoped by credentials; they do not take a sub-account header.

## Endpoints

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `list` | GET | `/v1/proceeds` | `ListProceeds` |
| `get` | GET | `/v1/proceeds/{id}` | `GetProceeds` |

## Verbs supported

- `list` — list proceeds, cursor-paginated.
- `get <id>` — fetch one proceeds record.

## Resource-specific flags

**`list` filters:** `--created-after` → `created_after`; `--created-before` → `created_before`; `--deposits-after` → `deposits_after`; `--deposits-before` → `deposits_before`.

## Table columns

**`list`:** `id`, `amount` (rendered `10.00 USD`), `status`, `deposits_at`, `created_at`.

**`get`:** key/value of the full proceeds record. Notable fields: `id`, `amount`, `currency` (`usd`, `cad`), `status` (`scheduled`, `paid`, `failed`, `pending`, `in_transit`, `canceled`), `payout_type` (`proceeds`), `payments_total`, `refunds_total`, `deposits_at`, `created_at`.

## References

- JustiFi Proceeds API: https://docs.justifi.tech/api-spec (Proceeds tag)
- [resource-command-template.md](resource-command-template.md)
