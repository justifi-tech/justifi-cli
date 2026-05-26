# Entities

| | |
|--------|----------------------------------------------|
| Status | Ready |
| Last Updated | 2026-05-26 |

## Overview

`justifi entities` manages the onboarding building blocks used to provision a merchant: businesses, identities, addresses, documents, entity bank accounts, terms-and-conditions acceptance, and product provisioning. Maps to the JustiFi **Entity Resources** tag group. The full onboarding flow is composed from these sub-resources (create a business → add a bank account → upload documents → accept terms → provision), then the resulting sub-account's status is read via [sub-accounts.md](sub-accounts.md).

Each sub-resource is a nested command group: `justifi entities <sub-resource> <verb>`.

Inherits all shared command behavior from [resource-command-template.md](resource-command-template.md).

**Sub-account scoping:** entity endpoints are scoped by credentials; they do not take a sub-account header.

## Sub-resources and endpoints

### `business`

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `list` | GET | `/v1/entities/business` | `ListBusinesses` |
| `get` | GET | `/v1/entities/business/{id}` | `GetBusiness` |
| `create` | POST | `/v1/entities/business` | `CreateBusiness` |
| `update` | PATCH | `/v1/entities/business/{id}` | `UpdateBusiness` |

Key create/update fields: `legal_name`, `doing_business_as`, `website_url`, `email`, `phone`, `classification` (`government`, `limited`, `non_profit`, `partnership`, `corporation`, `public_company`, `sole_proprietor`), `industry`, `mcc`, `tax_id`, `date_of_incorporation`, `country_of_establishment` (`USA`, `CAN`), `legal_address` (Address object or id), `representative` (Identity object or id), `owners` (up to four), `metadata`. `update` additionally accepts `archived`. List filter: `--business-name` → `business_name`. Table: `id`, `legal_name`, `email`, `created_at`.

### `identity`

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `list` | GET | `/v1/entities/identity` | `ListIdentities` |
| `get` | GET | `/v1/entities/identity/{id}` | `GetIdentity` |
| `create` | POST | `/v1/entities/identity` | `CreateIdentity` |
| `update` | PATCH | `/v1/entities/identity/{id}` | `UpdateIdentity` |

Key fields: `name`, `title`, `email`, `phone`, `dob_day`, `dob_month`, `dob_year`, `identification_number`, `is_owner`, `address` (Address object or id, create only), `metadata`. Responses return `ssn_last4`. Table: `id`, `name`, `email`, `is_owner`, `created_at`.

### `address`

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `list` | GET | `/v1/entities/address` | `ListAddresses` |
| `get` | GET | `/v1/entities/address/{id}` | `GetAddress` |
| `create` | POST | `/v1/entities/address` | `CreateAddress` |
| `update` | PATCH | `/v1/entities/address/{id}` | `UpdateAddress` |

Fields: `line1`, `line2`, `city`, `state`, `postal_code`, `country`. Table: `id`, `line1`, `city`, `state`, `postal_code`.

### `document`

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `list` | GET | `/v1/entities/document` | `ListDocuments` |
| `get` | GET | `/v1/entities/document/{id}` | `GetDocument` |
| `create` | POST | `/v1/entities/document` | `CreateDocument` |

Create fields: `file_name`, `file_type`, `document_type` (enum incl. `articles_of_incorporation`, `bank_statement`, `driver_license`, `passport`, `voided_check`, `other`, …), one of `business_id` / `identity_id` (required), `description`, `metadata`. `create` returns a presigned upload URL; `get` returns a presigned download URL. Document `status`: `pending`, `uploaded`, `canceled`. Table: `id`, `file_name`, `document_type`, `status`, `created_at`.

### `bank-accounts`

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `list` | GET | `/v1/entities/bank_accounts` | `ListBankAccounts` |
| `get` | GET | `/v1/entities/bank_accounts/{id}` | `GetBankAccount` |
| `create` | POST | `/v1/entities/bank_accounts` | `CreateBankAccount` |

Create fields (required): `account_owner_name`, `account_type` (`checking`, `savings`), `account_number`, `routing_number`, `business_id`, `bank_name`; optional `nickname`, `metadata`. Responses return `acct_last_four`, `country` (`US`, `CA`), `currency` (`usd`, `cad`). List filter: `--business-id` → `business_id`. These are entity/payout bank accounts, separate from payment-method bank accounts. Table: `id`, `bank_name`, `acct_last_four`, `account_type`, `created_at`.

### `terms-and-conditions`

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `accept` | POST | `/v1/entities/terms_and_conditions` | `TermsAndConditions` |

Records terms acceptance. Fields (required): `business_id`, `accepted`, `ip`; optional `user_agent`.

### `provisioning`

| Verb | Method | Path | operationId |
|------|--------|------|-------------|
| `create` | POST | `/v1/entities/provisioning` | `ProductProvisioning` |

Provisions a business onto a product, creating the associated sub-account. Fields (required): `business_id`, `product_category` (e.g. `payment`). Response returns `account_type`, `sub_account_id`, `platform_account_id`.

## Special behavior

- **`document create` is a two-step upload.** The endpoint returns a presigned upload URL; the CLI uploads the local file (supplied via `--file`) to that URL and reports success once complete. `document get` returns a presigned download URL the CLI can fetch with `--output-file`.
- **Object-or-id fields.** `business` accepts a nested `legal_address`/`representative`/`owners` or references to existing entity ids. The CLI exposes the id form via flags (`--legal-address-id`, `--representative-id`, `--owner-id` repeatable); full nested-object creation is done by creating those entities first.

## References

- JustiFi Entities / Onboarding API: https://docs.justifi.tech/api-spec (Entity Resources)
- [sub-accounts.md](sub-accounts.md) — read provisioned sub-account status
- [resource-command-template.md](resource-command-template.md)
