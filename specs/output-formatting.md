# Output Formatting

| | |
|--------|----------------------------------------------|
| Status | Ready |
| Last Updated | 2026-05-26 |

## Overview

How the CLI renders results to stdout/stderr, how it chooses between human-friendly and machine-friendly formats, and what exit codes it uses. The format auto-detects: when stdout is a terminal (TTY), the CLI renders a compact table. When stdout is piped, redirected, or otherwise not a TTY (which is how AI tools invariably invoke CLIs), it emits JSON. Flags can force either mode.

**Scope:**
- Output format selection (auto / `--json` / `--output`)
- JSON schema for success, error, and paginated responses
- Table rendering rules for humans
- Exit code mapping
- Logging vs result output (stderr vs stdout)
- Pagination output

**Out of scope:**
- HTTP-layer errors and retries — see [http-client.md](http-client.md)
- Per-resource output shape (which fields appear in tables) — see resource specs and [resource-command-template.md](resource-command-template.md)
- Credentials and auth errors — see [config-and-auth.md](config-and-auth.md)

**Dependencies:** [foundation.md](foundation.md) for the global flags.

**Design principles:**
- **Pipes get JSON, terminals get tables.** AI tools and shell scripts almost always pipe; humans almost always don't. The CLI must do the right thing without a flag.
- **JSON shape mirrors the JustiFi API response shape.** Don't invent a new schema; pass through what the API returned plus a thin envelope when needed.
- **Errors are JSON when JSON is the output mode.** AI tools must be able to parse error responses, not regex stderr.
- **Exit codes are stable and few.** AI tools branch on them; humans rarely see them. Don't proliferate.

## Specification

### Output mode resolution

For each invocation, the active mode is resolved in this order (first match wins):

1. `--output <format>` flag, where `<format>` is one of `json`, `table`, `auto`. Explicit override.
2. `--json` flag — shorthand for `--output json`.
3. `JUSTIFI_OUTPUT` env var with values `json`, `table`, or `auto`.
4. If stdout is a TTY → `table`.
5. If stdout is not a TTY → `json`.

`--json` and `--output table` together are a usage error (exit `2`).

### JSON output schema

**Success envelope (single resource):**

```json
{
  "data": { "id": "py_abc...", "amount": 1000, "currency": "usd", "...": "..." }
}
```

`data` contains the JustiFi API response object verbatim — same field names, same types. The CLI does not rename or restructure resource fields.

**Success envelope (list / paginated):**

```json
{
  "data": [ { "id": "py_..." }, { "id": "py_..." } ],
  "page_info": {
    "has_next":     true,
    "has_previous": false,
    "start_cursor": "abc...",
    "end_cursor":   "xyz..."
  }
}
```

`page_info` is the JustiFi cursor pagination metadata. When the underlying API call returns `page_info`, the CLI passes it through. Commands that do not paginate omit the field entirely.

**Success envelope (action with no body):**

```json
{ "data": null, "action": "voided", "resource_id": "py_abc..." }
```

Used for endpoints whose only meaningful result is a state change (e.g. `payments void`). `action` is the lowercase verb just performed.

**Error envelope:**

```json
{
  "error": {
    "code":    "card_declined",
    "message": "The card was declined.",
    "source":  "justifi_api"
  },
  "request_id": "req_..."
}
```

- `code` mirrors the JustiFi API error code where present. CLI-originated errors use codes from a fixed set defined below.
- `message` is human-readable. The CLI does not localize.
- `source` is one of: `justifi_api` (passed through), `cli_usage` (user error before any HTTP call), `cli_config` (credential / config issue), `cli_network` (transport failure), `cli_internal` (bug).
- `request_id` is the JustiFi request id from the response header when available; omitted otherwise.

**CLI-originated error codes (`source` ≠ `justifi_api`):**

| Code | Meaning |
|------|---------|
| `missing_credentials` | No credentials resolved (see [config-and-auth.md](config-and-auth.md)) |
| `invalid_credentials` | JustiFi rejected the credentials at the token endpoint |
| `invalid_flags` | Mutually exclusive or malformed flags |
| `invalid_input` | A required flag or argument is missing or invalid |
| `config_file_unreadable` | Config file path resolved but read failed |
| `config_file_permissions` | Config file has unsafe permissions (broader than `0600`) |
| `network_error` | DNS / TCP / TLS failure |
| `timeout` | Request exceeded `--timeout` |
| `rate_limited` | JustiFi returned `gateway_rate_limit_error` after retries exhausted |
| `internal_error` | Unexpected internal failure (panic, encoding bug) |

### Table output

Human table output is rendered when `mode == table`.

**Rules:**
- Each command declares the field set its table form shows. Tables are intentionally narrower than full JSON — they show only "what a human scanning the list needs". The per-resource specs name the exact columns.
- Column widths auto-size to terminal width, with the last column truncated with an ellipsis (`…`) when needed.
- IDs are not truncated (they are the column most often copy-pasted).
- Monetary amounts render with currency: `10.00 USD` not `1000` cents. This is a human-readability concession; JSON output passes the raw integer.
- Timestamps render as RFC 3339 in the user's local timezone in tables; UTC ISO 8601 in JSON.
- For a single-resource fetch (`justifi payments get <id>`), the table form is a two-column "key / value" pair, one row per field shown.
- An empty result set prints `(no results)` to stderr and exits `0`.

**Color and styling:**
- Color is on by default for table output to a TTY. Disabled when:
  - `NO_COLOR` env var is set (any value).
  - `--no-color` flag is set.
  - Stdout is not a TTY.
- Only used for: header row (bold), error labels (red), and warning labels (yellow). No other ornamentation.

### Pagination output

For list commands:
- The CLI passes through one page at a time. It does **not** auto-paginate.
- `--limit N` (default 25, max 100) maps directly to the JustiFi `limit` query param.
- `--after <cursor>` and `--before <cursor>` map to `after_cursor` / `before_cursor`.
- In JSON mode, `page_info` is emitted in the envelope so the caller can issue the next request.
- In table mode, after the table, the CLI prints a one-line hint to stderr when `has_next` is true: `more results: use --after <cursor> to continue` with the actual cursor. Stderr (not stdout) so it does not contaminate piped output.
- A `--all` flag may be added in a future spec to auto-paginate; not in v1.

### Stdout vs stderr

- **Stdout** carries only the command's result (table or JSON).
- **Stderr** carries: log messages, hints (`more results: ...`), warnings, and error envelopes when the mode is `table`. When the mode is `json`, error envelopes go to **stdout** so a caller's pipe captures them, and exit code signals failure.
- `--verbose` / `-v` increases stderr log verbosity (request/response summaries, retry attempts).
- `--quiet` / `-q` suppresses all stderr output except final error messages.
- `--verbose` and `--quiet` together are a usage error (exit `2`).

### Exit codes

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | JustiFi API returned an error (any 4xx/5xx after retries) or auth rejected by JustiFi |
| `2` | CLI usage error: bad flags, missing required args, missing credentials, config file unreadable, mutually exclusive flags |
| `3` | Network / transport failure: DNS, TCP, TLS, timeout |
| `4` | Rate limited after retry budget exhausted |
| `70` | Internal CLI error (panic, encoding bug). Reserved range `70`. |
| `130` | Interrupted by signal (Ctrl-C, SIGINT) |

Exit codes `5`–`69` and `71`–`129` are reserved for future use; commands MUST NOT emit codes outside the table above.

### Output configuration for AI tools

When invoked by an AI agent, the CLI should be 100% non-interactive and machine-parseable:
- Stdout JSON is fully parseable as a single JSON object (no banners, no progress indicators interleaved).
- Progress indicators (spinners, percent bars) are TTY-only. They are written to stderr and only when stdout AND stderr are both TTYs.
- Long-running operations (e.g. report download) stream to stdout in `table` mode but emit a single final JSON object in `json` mode (after the operation completes). Streaming JSON is not used in v1; revisit if a use case appears.

## Notes

- **Why mirror the JustiFi response shape in JSON output.** AI tools and customers learning the API can compare `justifi payments get <id>` output to the JustiFi API docs field-for-field. Inventing a new schema would create a second thing to learn.
- **Why an envelope (`data`/`error`) at all.** Without it, a list response would be a bare JSON array, which makes it impossible to attach `page_info` or `request_id` without restructuring. The envelope is the cheapest stable shape.
- **Why hint on stderr instead of stdout.** A caller piping `justifi payments list --json | jq …` would otherwise have to filter the hint out, defeating the point.
- **Why exit code 4 for rate-limited.** AI tools and ops scripts often want to back off and retry on rate limits specifically, without doing string matching on the error code. A dedicated exit code is the easiest signal.

## References

- [JustiFi API errors](https://docs.justifi.tech/api-spec) — error code shape we mirror.
- [NO_COLOR convention](https://no-color.org/)
- POSIX exit-code conventions; sysexits ranges (we deliberately deviate by keeping the set small).
