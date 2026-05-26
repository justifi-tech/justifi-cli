# Resource Command Template

| | |
|--------|----------------------------------------------|
| Status | Ready |
| Last Updated | 2026-05-26 |

## Overview

The shared contract every `justifi <resource> <verb>` command follows. Resource specs ([payments.md](payments.md), [refunds.md](refunds.md), etc.) reference this template for everything that does not vary by resource — verb naming, shared flags, output shape, error behavior. Resource specs only need to enumerate which verbs they support, which fields are arguments vs flags, and which columns show up in their table view.

**Scope:**
- Verb vocabulary (`list`, `get`, `create`, `update`, `delete`, custom action verbs)
- Shared flags (those that appear on most or all commands)
- Argument vs flag conventions
- Output shape (what the JSON envelope and table columns look like)
- Error and exit-code behavior at the command boundary

**Out of scope:**
- Per-resource verb sets, fields, and columns — that's the job of each resource spec.
- Output format mechanics — see [output-formatting.md](output-formatting.md).
- HTTP client behavior — see [http-client.md](http-client.md).
- Credentials and sub-account scoping — see [config-and-auth.md](config-and-auth.md).

**Dependencies:** [foundation.md](foundation.md), [config-and-auth.md](config-and-auth.md), [output-formatting.md](output-formatting.md), [http-client.md](http-client.md).

**Design principles:**
- **Predictability over cleverness.** A user who knows `justifi payments list` should be able to guess `justifi refunds list` without reading the docs.
- **One way to do a thing.** No aliases for verbs (no `ls` for `list`, no `show` for `get`). Consistency is more valuable than brevity.
- **Per-resource specs are short by default.** If a resource spec exceeds two pages, it is either covering an exotic resource or repeating content this template already provides.

## Specification

### Verb vocabulary

The CLI uses a fixed verb set. Resource specs select from this list; they do not invent new verbs unless they document the new verb in this section first.

| Verb | Maps to | Notes |
|------|---------|-------|
| `list` | `GET /<resource>` | Cursor-paginated; never auto-pages |
| `get <id>` | `GET /<resource>/<id>` | Single-resource fetch |
| `create` | `POST /<resource>` | Body via flags; idempotency key required |
| `update <id>` | `PATCH /<resource>/<id>` | Partial update; body via flags |
| `delete <id>` | `DELETE /<resource>/<id>` | Where the API supports it; rare for JustiFi resources |

**Approved custom action verbs** (used only when a resource spec explicitly calls them out):
- `capture <id>` — confirm an authorized payment (payments)
- `void <id>` — cancel an authorized, uncaptured payment (payments)
- `refund <id>` — initiate a refund against a payment (payments)
- `balance-transactions <id>` — list a payment's ledger entries (payments)
- `clone <token>` — copy a payment method to another account (payment-methods)
- `remove-payment-method <id> <payment-method-id>` — remove a member from a group (payment-method-groups)
- `add-evidence <id>`, `update-response <id>`, `submit-response <id>` — dispute evidence and response (disputes)
- `complete <id>` — complete a checkout (checkouts)
- `settings <id>`, `payout-account <id>` — sub-account sub-reads (sub-accounts)
- `scheduled`, `history <fee-type>` — fee-configuration variants (fee-configurations)
- `accept` — record terms acceptance (entities terms-and-conditions)
- `provision`/`create` — provision a business onto a product (entities provisioning)
- `status <id>`, `identify <id>` — terminal device actions (terminals)

New custom verbs require an amendment to this list. Adding `--operation foo` to an existing verb is preferred over inventing a new verb whenever possible.

### Argument vs flag conventions

- **Positional argument:** the resource identifier (`<id>`) for any verb that targets one resource.
- **Flags:** every other field. No positional arguments beyond the id.
- Booleans use `--<name>` for true, `--<name>=false` for false. Never `--no-<name>` (cobra supports it but it inverts intuition for some flags).
- Money is passed as an integer in the smallest currency unit (cents), via `--amount`. The output renders it as `10.00 USD` in table mode and as the raw integer in JSON mode (see [output-formatting.md](output-formatting.md)).
- Currency codes are lowercase three-letter ISO 4217 (`usd`, `cad`).
- Time arguments accept either RFC 3339 (`2026-05-26T12:00:00Z`) or relative (`-7d`, `-24h`).
- IDs follow JustiFi's prefix convention (e.g. `py_`, `sa_`). The CLI does not validate prefixes; it forwards whatever was provided.

### Globally-available flags

Every resource command inherits the global flag surface defined in [foundation.md](foundation.md#command-framework). Resource specs do not restate those flags.

### Flags common to specific verbs

**`list` verb** — every list command inherits:

| Flag | Type | Default | Purpose |
|------|------|---------|---------|
| `--limit` | int | 25 | Page size, max 100 |
| `--after <cursor>` | string | — | Page forward |
| `--before <cursor>` | string | — | Page backward |
| `--created-after <time>` | string | — | Filter by `created_at >= time`, when the resource supports it |
| `--created-before <time>` | string | — | Filter by `created_at < time`, when the resource supports it |

Resource-specific filter flags (e.g. `--status`, `--payment-method-type`) are declared in the per-resource spec. They map directly to JustiFi query params with kebab-case → snake_case conversion.

**`create` / `update` / custom-mutating verbs** — every mutating command inherits:

| Flag | Type | Default | Purpose |
|------|------|---------|---------|
| `--idempotency-key <string>` | string | auto-generated when omitted | Idempotency-Key header; any string accepted ([http-client.md](http-client.md)) |
| `--metadata <key=value>` | repeatable | — | Sets the request body's metadata map, where the resource supports one. On update verbs the supplied set replaces the resource's existing metadata. |

A resource spec lists only its resource-specific flags. It does not restate `--idempotency-key` or `--metadata`; a mutating verb is understood to accept them. When a resource's metadata support or replace behavior differs from the above, the resource spec notes only the difference.

### Output shape

**For `list`:**
- JSON: `{ "data": [...], "page_info": {...} }` (per [output-formatting.md](output-formatting.md)).
- Table: header row + one row per item. Resource spec names the columns.

**For `get`, `create`, `update`:**
- JSON: `{ "data": {...} }`.
- Table: two-column key/value rendering. Resource spec names which keys appear, in what order. Default order: `id`, then alphabetical, with `created_at`/`updated_at` last.

**For custom action verbs that mutate state but return no body of interest:**
- JSON: `{ "data": null, "action": "<verb>", "resource_id": "<id>" }`.
- Table: a single line `<verb> <id> ok`.

### `--help` and Long descriptions

Every command (resource parent and verb subcommand) must satisfy:

1. **`Short`**: a single sentence, ≤80 chars, imperative voice, ends without a period. Example: `List payments`.
2. **`Long`**: at least three sentences covering (a) what the command does, (b) what JustiFi endpoint it calls, (c) which sub-account scoping applies.
3. **Examples**: at least one `cobra.Example` block per verb, ideally two — a happy path and a common variant (e.g. with `--json` or filtering).
4. **Flag descriptions**: every flag, including inherited ones, must show in `--help` with a non-empty description.

**Example for `payments list`:**

```text
List payments

Usage:
  justifi payments list [flags]

Long:
  List payments associated with the active sub-account. Calls
  GET /v1/payments. Requires a sub-account either via --sub-account, the
  active profile, or JUSTIFI_SUB_ACCOUNT.

Examples:
  # First page in table form (default for a terminal)
  justifi payments list --limit 10

  # JSON output piped to jq, last 24h
  justifi payments list --json --created-after -24h | jq '.data[].id'

Flags:
  --limit int            page size (1-100) (default 25)
  --after string         cursor returned by previous page
  --created-after time   filter created_at >= time (RFC 3339 or relative like -7d)
  --status string        filter by status (succeeded, failed, pending)
  ... (all inherited flags also listed) ...
```

### Resource parent command

Every resource group registers a parent command (e.g. `paymentsCmd`) that is itself a no-op — it only exists to be the cobra parent for the verbs. Running `justifi payments` with no verb prints the help and exits `2`. Running `justifi payments --help` prints help and exits `0`.

### Errors at the command boundary

Commands return errors from `RunE`. The root command's error handler:
- Maps `client` errors (see [http-client.md](http-client.md)) to the appropriate JSON envelope and exit code.
- Calls `cobra.Command.Usage()` only for `invalid_flags` and `invalid_input` errors — never for API or network errors.
- Treats panics as exit code `70` with the panic message redacted to a generic "internal error" in non-verbose mode, and the full stack trace at `--verbose`.

### Sub-account requirement

Most JustiFi endpoints require a sub-account. Each resource spec must state, in a single sentence, whether the resource's endpoints require, accept, or ignore the sub-account header. The CLI:
- Errors fast with exit `2` if a resource requires a sub-account and none is resolved.
- Sends the header when one is resolved, regardless of whether the endpoint uses it (JustiFi will ignore extraneous headers).

### Testing

This is a thin command layer; testing follows the policy in [foundation.md](foundation.md#test-layering):
- **Validation and orchestration are tested directly** — the resource-specific branching that can actually be wrong (mutual exclusions, required-flag/sub-account errors, enum validation, file upload/download). Where this logic can be a pure function, it is tested as one with no client involved.
- **A consumer-defined gomock mock is used only** when a test needs a canned client response to drive orchestration (e.g. the document two-step upload, `reports --download`). The consuming command package declares the narrow interface it needs and generates the mock beside it; the mock is never asserted against the real client.
- **Passthrough verbs are not unit-tested** — list/get/create that only call the client and render carry no logic to test. They are covered by the sandbox smoke suite.
- **Output and config are tested in their own packages**, never re-tested per command.

### Adding a new resource — checklist

Authoring a new resource spec is a small, scriptable job once this template is in hand. Each resource spec must include:

1. **Overview** — one paragraph: what the resource is, which JustiFi tag it maps to, sub-account scoping behavior.
2. **Endpoints** — table of `(verb, HTTP method, path, operationId)`.
3. **Verbs supported** — list with one-line purpose each.
4. **Resource-specific flags** — table per verb, only the flags not in the global/common sets.
5. **Table columns** — for `list` and `get`, name the columns shown.
6. **Special behavior** — anything not covered by this template (long-running operations, file output, etc.).
7. **References** — link to JustiFi API docs page for this resource.

Resource specs should fit on two pages. If a resource spec exceeds two pages, the resource is genuinely complex (e.g. `entities` with its sub-resources) or the template needs an update.

## Notes

- **Why no verb aliases.** Aliases double the surface for tab-completion, docs, and AI tool discovery. The win (`ls` vs `list`) is small; the cost is real.
- **Why money in integers via `--amount`.** Matches how JustiFi serializes amounts. Avoids floating-point ambiguity at the flag layer. The output formatter handles the human-friendly form.
- **Why the resource parent is a no-op.** Mirrors cobra convention used by `kubectl`, `gh`, and similar tools. It would be tempting to make `justifi payments` an alias for `justifi payments list`, but ambiguity (which list? for which sub-account?) outweighs the brevity gain.
- **Why ≤2 pages per resource spec.** Pages of repetition across 17 resource specs would be a maintenance liability and would dilute the per-resource information that actually matters. The template absorbs the shared content.

## References

- [foundation.md](foundation.md), [config-and-auth.md](config-and-auth.md), [output-formatting.md](output-formatting.md), [http-client.md](http-client.md)
- JustiFi API reference: https://docs.justifi.tech/api-spec
