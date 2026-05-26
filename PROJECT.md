# justifi-cli

## Build & Test

- `make test` runs unit tests; `make test-all` adds the sandbox smoke suite. Validate with the full suite — if a sandbox/integration test fails, that's a real failure, never dismissed.
- This is a thin CLI: most commands parse flags, make one API call, and render. **Test where the logic is, not in every command.** Concentrate tests on `internal/client` (against `httptest.Server`), `internal/output`, and the api-sync scripts. In the command layer, test only resource-specific validation/orchestration. Keep `internal/config` tests minimal (resolution-order precedence and token-cache validity only) — excessive config testing is unwanted.
- **Do not test passthrough.** A mock-based test of a list/get/create that just calls the client and renders verifies only the plumbing and breaks on refactors. Cover passthrough with the sandbox smoke suite instead.
- Shared tools (`internal/output`, `internal/config`) are tested once in their own package; commands never re-test formatting or config.

## Conventions

- Follow patterns in existing code for naming, structure, and style
- Match the language and framework conventions of the project
<!-- Customize: add language-specific conventions here -->

### Interfaces and Dependencies

- **Interfaces are defined by the consumer.** The package that uses a dependency declares the narrow interface it needs (only the methods it calls). Provider packages return concrete types.
- **Dependency injection over globals.** Pass dependencies explicitly rather than importing singletons.

### Mocks and seams

- **HTTP/API clients are concrete, with no interface.** Fake them at the `http.RoundTripper` transport seam — inject a `*http.Client` with a stub transport (or an `httptest.Server`) and run the real client. This is how both client and command-orchestration tests work.
- **gomock + consumer-defined interfaces are only for non-HTTP behavioral seams** (interactive prompts, persistent config, OS actions). When one exists: the consumer declares the interface, and the mock is generated next to it with gomock — never handwritten.
- **Never write a test that tests a mock.** Asserting a mock returns what it was told to return verifies nothing, and no mock-vs-real contract test is needed (generated mocks can't drift).
- A real implementation is tested directly (e.g. the HTTP client against `httptest.Server`), never via its own mock.
