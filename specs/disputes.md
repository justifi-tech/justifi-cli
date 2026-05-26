# Disputes

| | |
|--------|----------------------------------------------|
| Status | Ready |
| Last Updated | 2026-05-26 |

## Overview

`justifi disputes` manages the chargeback lifecycle: listing and retrieving disputes, submitting evidence files, and building and submitting a dispute response. Maps to the JustiFi **Disputes** tag.

Inherits all shared command behavior from [resource-command-template.md](resource-command-template.md).

**Sub-account scoping:** `list` accepts a sub-account to scope results. The remaining verbs operate by dispute id.

## Endpoints

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `list` | GET | `/v1/disputes` | `ListDisputes` |
| `get` | GET | `/v1/disputes/{id}` | `GetDispute` |
| `update` | PATCH | `/v1/disputes/{id}` | `UpdateDispute` |
| `add-evidence` | PUT | `/v1/disputes/{id}/evidence` | `CreateDisputeEvidence` |
| `update-response` | PATCH | `/v1/disputes/{id}/response` | `UpdateDisputeResponse` |
| `submit-response` | POST | `/v1/disputes/{id}/response` | `SubmitDisputeResponse` |

## Verbs supported

- `list` — list disputes, cursor-paginated.
- `get <id>` — fetch one dispute.
- `update <id>` — replace a dispute's metadata.
- `add-evidence <id>` — register an evidence file and upload it.
- `update-response <id>` — set the textual fields of the dispute response (draft).
- `submit-response <id>` — finalize and submit the dispute response, or forfeit.

## Resource-specific flags

**`update <id>`:** metadata only, via the inherited `--metadata` flag.

**`add-evidence <id>`:**

| Flag | Maps to | Notes |
|------|---------|-------|
| `--file` | `file_name` + upload | Local path to the evidence file; the file name and type derive from it |
| `--evidence-type` | `dispute_evidence_type` | One of `cancellation_policy`, `customer_communication`, `customer_signature`, `duplicate_charge_documentation`, `receipt`, `refund_policy`, `service_documentation`, `shipping_documentation`, `uncategorized_file` |
| `--description` | `description` | |

Supported file types: `image/jpeg`, `image/png`, `application/pdf`, `application/zip`.

**`update-response <id>` / `submit-response <id>`:** one flag per response field, including `--additional-statement`, `--cancellation-policy-disclosure`, `--cancellation-rebuttal`, `--customer-billing-address`, `--customer-email-address`, `--customer-name`, `--customer-purchase-ip-address`, `--duplicate-charge-explanation`, `--duplicate-charge-original-payment-id`, `--product-description`, `--refund-policy-disclosure`, `--refund-refusal-explanation`, `--service-date`, `--shipping-address`, `--shipping-carrier`, `--shipping-date`, `--shipping-tracking-number`.

`submit-response` additionally accepts `--forfeit` → `forfeit` (boolean). When `--forfeit` is set, the response is forfeited and the other fields are ignored.

## Table columns

**`list`:** `id`, `payment_id`, `amount` (rendered `10.00 USD`), `currency`, `status`, `reason`, `due_date`.

**`get`:** key/value of the full dispute. Notable fields: `id`, `payment_id`, `account_id`, `amount`, `currency` (`usd`, `cad`), `reason`, `due_date`, `status` (`needs_response`, `under_review`, `won`, `lost`), `dispute_response`, `dispute_reversal`, `created_at`, `updated_at`.

## Special behavior

- **`add-evidence` is a two-step upload.** The JustiFi endpoint returns a presigned upload URL; the CLI uploads the local `--file` contents to that URL and reports success once the upload completes.

## References

- JustiFi Disputes API: https://docs.justifi.tech/api-spec (Disputes tag)
- [resource-command-template.md](resource-command-template.md)
