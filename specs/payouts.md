# Payouts

| | |
|--------|----------------------------------------------|
| Status | Ready |
| Last Updated | 2026-05-26 |

## Overview

`justifi payouts` lists and retrieves settlement payouts and updates their metadata. Maps to the JustiFi **Payouts** tag.

Inherits all shared command behavior from [resource-command-template.md](resource-command-template.md).

**Sub-account scoping:** payouts endpoints are scoped by credentials; they do not take a sub-account header.

## Endpoints

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `list` | GET | `/v1/payouts` | `ListPayouts` |
| `get` | GET | `/v1/payouts/{id}` | `GetPayout` |
| `update` | PATCH | `/v1/payouts/{id}` | `UpdatePayout` |

## Verbs supported

- `list` — list payouts, cursor-paginated.
- `get <id>` — fetch one payout.
- `update <id>` — replace a payout's metadata.

## Resource-specific flags

**`update`:** carries no resource-specific flags; it updates metadata via the inherited `--metadata` flag.

**`list` filters:** `--created-after` → `created_after`; `--created-before` → `created_before`; `--deposits-after` → `deposits_after`; `--deposits-before` → `deposits_before`.

## Table columns

**`list`:** `id`, `amount` (rendered `10.00 USD`), `status`, `payout_type`, `deposits_at`, `created_at`.

**`get`:** key/value of the full payout. Notable fields: `id`, `amount`, `currency` (`usd`, `cad`), `status` (`paid`, `failed`, `forwarded`, `scheduled`, `in_transit`, `canceled`), `payout_type` (`ach`, `cc`), `fees_total`, `payments_total`, `deposits_at`, `created_at`.

## References

- JustiFi Payouts API: https://docs.justifi.tech/api-spec (Payouts tag)
- [resource-command-template.md](resource-command-template.md)
