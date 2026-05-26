# HTTP Client

| | |
|--------|----------------------------------------------|
| Status | Ready |
| Last Updated | 2026-05-26 |

## Overview

How the CLI talks to the JustiFi REST API. The CLI uses its own HTTP client built on the standard library — it does **not** depend on the official `justifi-go` SDK. Request and response shapes come from the generated `justifiapi` types (see [api-sync.md](api-sync.md)); the client layer adds authentication, retries, idempotency, the `Sub-Account` header, and error translation on top of `net/http`. All command code reaches the API through an internal `Client` interface so that commands are testable without network access.

**Scope:**
- The `internal/client.Client` interface and its sub-clients
- The standard-library HTTP implementation behind it (auth, transport, retries, headers)
- How generated types from [api-sync.md](api-sync.md) plug in
- Retries, idempotency, pagination passthrough, timeouts, logging
- Error translation to the CLI's exit-code surface

**Out of scope:**
- Credential resolution and token caching — see [config-and-auth.md](config-and-auth.md)
- Output rendering and exit codes — see [output-formatting.md](output-formatting.md)
- Individual resource commands — see resource specs
- The OpenAPI sync pipeline that produces request/response types — see [api-sync.md](api-sync.md)

**Dependencies:**
- [config-and-auth.md](config-and-auth.md) — supplies credentials, `api_base`, and the active sub-account.
- [output-formatting.md](output-formatting.md) — consumes errors from this layer.
- [api-sync.md](api-sync.md) — provides generated types in `internal/justifiapi/`.

**Design principles:**
- **No SDK dependency.** The CLI owns its HTTP layer end-to-end. This decouples CLI velocity from the cadence and quality of any external Go SDK and removes a class of version-skew and bug-inheritance problems.
- **Generated types, hand-written transport.** The request/response shapes are generated from the OpenAPI spec and refresh automatically via [api-sync.md](api-sync.md). The transport behavior (auth, retries, headers) is small, stable, and hand-written.
- **One interface, mockable everywhere.** Every command depends on the `Client` interface, never on the concrete HTTP implementation. Unit tests use a generated mock; the implementation has its own tests against a fake HTTP server.
- **Behavior over mechanism.** This spec defines observable behavior (what headers are sent, when retries happen, what errors mean). It does not prescribe internal type or file structure.

## Specification

### The client

`internal/client` exposes a **concrete** client built on the Go standard library (`net/http`). It returns concrete types and defines no interface — neither here nor for consumers to depend on. The testing seam is the `*http.Client` transport (see "Testing"), so no client interface is needed anywhere. The client grows one method per resource as resources are added; there is no parallel interface that could fall out of sync.

The client mirrors the JustiFi resource taxonomy: one accessor per resource group, each returning a concrete per-resource service. Conceptually:

```go
type Client struct { /* ... */ }

func (c *Client) Payments() *PaymentsService { /* ... */ }
func (c *Client) Refunds()  *RefundsService  { /* ... */ }
// ... one accessor per JustiFi resource group ...

func (c *Client) SubAccount() string                { /* ... */ }
func (c *Client) WithSubAccount(id string) *Client  { /* ... */ }
```

Each per-resource service exposes the verbs for its resource. Per-resource specs name the verbs; this spec defines the method shape:

```go
func (s *PaymentsService) List(ctx context.Context, params justifiapi.ListPaymentsParams) (*justifiapi.PaymentList, error)
func (s *PaymentsService) Get(ctx context.Context, id string) (*justifiapi.Payment, error)
func (s *PaymentsService) Create(ctx context.Context, body justifiapi.CreatePaymentRequest, opts ...RequestOption) (*justifiapi.Payment, error)
```

**Contract rules:**
- Request/response types come from the generated `justifiapi` types (see [api-sync.md](api-sync.md)). The client never declares parallel types for shapes that already exist in the OpenAPI spec.
- Every method takes `context.Context` first.
- Every mutating method takes `opts ...RequestOption` last. Defined options:
  - `WithIdempotencyKey(key string)` — sets the `Idempotency-Key` header for this call.
  - `WithSubAccountOverride(id string)` — overrides the client-level sub-account for this call only.
- Errors are always typed (see "Error model" below).

### HTTP implementation

The client issues requests with the Go standard library (`net/http`). Required behaviors:

- One client instance per process, safe for concurrent use.
- Base URL comes from the resolved `api_base` (see [config-and-auth.md](config-and-auth.md)).
- The bearer token comes from the CLI's token cache (see [config-and-auth.md](config-and-auth.md)); the client sends `Authorization: Bearer <token>` on every request. The client does not perform the OAuth exchange itself — it receives an already-valid token at construction. (Obtaining/refreshing the token is the config-and-auth layer's job.)
- `Content-Type: application/json` on requests with a body; request and response bodies are JSON, marshalled to/from the generated types.
- A `User-Agent` identifying the CLI and its version is sent on every request.
- The `Sub-Account` header is attached when a sub-account is resolved (client-level or per-call override).
- Request construction, retry, and logging are layered so that each concern is independently testable.

### Retries

The client retries transient failures.

**Policy:**
- Retry on `408 Request Timeout`, `429 Too Many Requests`, `500`, `502`, `503`, `504`. Do not retry on `400`, `401`, `403`, `404`, `409`, `422` — those are caller-correctable.
- Maximum 3 attempts (initial + 2 retries) unless overridden by `--retries N`.
- Backoff: exponential with jitter, starting at 500ms, doubling, capped at 8s. A `Retry-After` header, when present, overrides backoff (clamped to 30s).
- On `429` specifically: if all retries are exhausted, the final error surfaces with exit code `4` (rate limited) per [output-formatting.md](output-formatting.md).
- Non-idempotent methods (POST without an `Idempotency-Key` header) are retried on 5xx **only** when no response was received (connection-level failure). If a 5xx response was received, do not retry — return the error.

### Idempotency

- All `POST` / `PATCH` / `PUT` calls accept `WithIdempotencyKey(key string)` and surface a `--idempotency-key` flag at the command layer.
- The flag's value is any non-empty string (the JustiFi API treats the header as an opaque string; the CLI does not enforce a UUID format).
- When the user does not supply a key for a mutating call, the CLI generates a UUIDv7 as the default value and includes it. The generated key is logged at `--verbose`. The user can always override with `--idempotency-key`.
- Read-only calls (`GET`, and `DELETE` semantics that JustiFi treats as idempotent without an explicit key) do not set the header.

### Pagination

- Interface methods return the JustiFi response object as-is (e.g. `*PaymentList`), which includes the data slice and `page_info` (`has_next`, `has_previous`, `start_cursor`, `end_cursor`).
- No auto-pagination in the client layer; commands pass `--limit`, `--after`, `--before` through to the matching JustiFi query params and emit one page. See [output-formatting.md](output-formatting.md) for cursor-hint emission.

### Timeouts

- Per-request timeout default: 30s. Override with `--timeout <duration>` on any command.
- The CLI sets the deadline on the `context.Context` passed to client methods; the transport and retry loop honor it.
- Long-running operations (e.g. report generation) define their own timeout in the resource spec.

### Logging

- The `--verbose` flag (see [output-formatting.md](output-formatting.md)) enables a request/response logger.
- Verbose log lines go to stderr, one per request:
  ```
  [HTTP] POST /v1/payments → 200 (124ms) idempotency_key=01HXXXX request_id=req_abc
  ```
- Request and response bodies are not logged even at `--verbose`. A future `--debug` flag may add body logging; out of scope for v1.
- `Authorization` header values are never logged.

### Error model

The client returns typed errors: sentinel values for category checks plus a structured `APIError` carrying detail.

```go
type APIError struct {
    StatusCode int     // HTTP status, 0 if no response received
    Code       string  // JustiFi error code, e.g. "card_declined"; "" if network error
    Message    string  // human-readable
    RequestID  string  // JustiFi request id header, if present
    Retryable  bool    // true if the CLI exhausted retries on a retryable status
    Underlying error   // wrapped lower-level error (for errors.Is/As)
}

var (
    ErrUnauthorized = errors.New("unauthorized")
    ErrForbidden    = errors.New("forbidden")
    ErrNotFound     = errors.New("not found")
    ErrConflict     = errors.New("conflict")
    ErrValidation   = errors.New("validation failed")
    ErrRateLimited  = errors.New("rate limited")
    ErrServerError  = errors.New("server error")
    ErrNetwork      = errors.New("network error")
    ErrTimeout      = errors.New("timeout")
)
```

**Translation rules from HTTP responses:**
- `401` → `ErrUnauthorized` (in `APIError`), exit `1`.
- `403` → `ErrForbidden`, exit `1`.
- `404` → `ErrNotFound`, exit `1`.
- `409` → `ErrConflict`, exit `1`.
- `422` → `ErrValidation`, exit `1`. Field-level detail lives in `APIError.Message`.
- `429` after retries → `ErrRateLimited`, exit `4`.
- `5xx` after retries → `ErrServerError`, exit `1`.
- Network/DNS/TLS failure → `ErrNetwork`, exit `3`.
- Context deadline → `ErrTimeout`, exit `3`.

The JustiFi error response body (`code`, `message`) populates `APIError.Code` and `.Message`. The JustiFi request-id response header populates `.RequestID` so output envelopes can surface it.

Commands return these errors from `RunE`; the root command's error handler maps them to the JSON error envelope (see [output-formatting.md](output-formatting.md)) and the matching exit code.

### Testing

The client is concrete and exposes no interface; faking happens at the **HTTP transport seam**, not via a client interface. The constructor accepts a `*http.Client`, so a test injects one whose `Transport` is a stub `http.RoundTripper` (or points it at an `httptest.Server`).

- The client is tested **directly** against an `httptest.Server` — a real request/response round-trip — exercising retry behavior, idempotency-key injection, sub-account-header injection, pagination, and every error-translation branch. This is where the bulk of the app's testable logic lives. Runs under `make test` (no live network).
- The few command tests with real orchestration (e.g. the document two-step upload, `reports --download`) use the **same transport seam**: run the real command against the real client wired to a stub transport that returns canned responses. No client interface, no client mock.
- gomock + consumer-defined interfaces are reserved for genuine **non-HTTP behavioral seams** (interactive prompts, persistent config, OS actions). The CLI has none of those in v1 (it is non-interactive and config is minimal), so v1 ships no client mock.

## Notes

- **Why no SDK.** The official `justifi-go` SDK is currently stale and was a source of friction. More structurally, a CLI's HTTP needs are small (auth, retries, headers, JSON encode/decode) and stable, while the API surface churns — and that churn is already handled by generated types in [api-sync.md](api-sync.md). Owning a small transport layer is cheaper than absorbing an SDK's bugs and release cadence. The reference CLI we studied takes the same approach: its HTTP layer is bespoke and does not import the vendor's Go SDK.
- **Why generated types instead of fully generic parsing.** We could parse responses generically (path-addressed JSON) and avoid types entirely. We don't, because typed request/response structs give command authors and AI tools real Go signatures to work against, and the types refresh automatically via [api-sync.md](api-sync.md) so the maintenance cost is near zero.
- **Why no auto-paginate in v1.** Adds complexity to error handling and output streaming. Cursor hints in table mode and `page_info` in JSON mode cover the common case. A future `--all` flag is straightforward to add later.
- **Why context-deadline-based timeouts.** A `context.Context` deadline propagates correctly through the transport, the retry loop, and any future goroutines. A separate `http.Client.Timeout` would not.

## References

- [api-sync.md](api-sync.md) — provides the generated types this layer consumes.
- [config-and-auth.md](config-and-auth.md) — supplies the token and base URL this layer uses.
- [UUIDv7 (RFC 9562)](https://www.rfc-editor.org/rfc/rfc9562) — idempotency key generation.
