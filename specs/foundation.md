# Foundation

| | |
|--------|----------------------------------------------|
| Status | Ready |
| Last Updated | 2026-05-26 |

## Overview

`justifi` is a command-line tool that wraps the public JustiFi API. It is used internally by JustiFi engineering and ops and distributed to customers who integrate with the platform. It must be usable both by humans at a terminal and by AI coding agents (Claude Code, Codex, etc.) that shell out to it.

This foundation spec defines the constraints that every other spec builds on: where source lives, what command framework is used, which guardrails (lint, hooks, CI) enforce quality, and how releases are produced. It does not describe individual commands or their implementations.

**Scope:**
- Repository layout and module identity
- Command framework choice and global flag surface
- Guardrails: lint, formatting, hooks, conventional commits, CI
- Release model and supported platforms
- Test layering

**Out of scope:**
- Credentials, profiles — see [config-and-auth.md](config-and-auth.md)
- Output formatting, exit codes — see [output-formatting.md](output-formatting.md)
- HTTP client / SDK integration — see [http-client.md](http-client.md)
- OpenAPI sync — see [api-sync.md](api-sync.md)
- Any resource-specific command — see resource specs
- MCP server — handled by `github.com/justifi-tech/mcp-server`; outside this CLI

**Dependencies:** None — this is the keystone spec.

**Design principles:**
- **CLI lives in the repo root.** The only deliverable is one binary; an internal `cmd/<name>/` directory adds indirection without value.
- **AI-tool friendliness is a constraint, not a feature.** Stable flag names, predictable JSON output, complete help text.
- **Guardrails first.** Lint config, formatter, and hooks are present before any source is committed; nothing has to be retrofitted to comply.
- **Inherit from `~/personal/go-blueprint`.** Where the blueprint's build/lint/test conventions apply to a CLI, this repo adopts them. Where the blueprint targets HTTP services (internal API package, chi router, OpenAPI generation for handlers, DB migrations), those parts are dropped.

## Specification

### Module identity

| | |
|---|---|
| Module path | `github.com/justifi-tech/justifi-cli` |
| Binary name | `justifi` |
| Go version | **1.26**, pinned in `go.mod`. Bumps require an explicit spec amendment. |

### Repository layout

The repository is a single Go module rooted at the path above. Application source lives in the repo root. Reusable internal code lives under `internal/`. The CLI does not expose any importable packages to external consumers.

Top-level layout:

| Path | Purpose |
|------|---------|
| `main.go`, `root.go`, `version.go` | Entry point, root cobra command, version subcommand |
| `<resource>.go` (one per resource group) | Resource parent command + verbs, package `main` |
| `internal/client/` | API client interface (see [http-client.md](http-client.md)) |
| `internal/config/` | Config + credential resolution (see [config-and-auth.md](config-and-auth.md)) |
| `internal/output/` | Output rendering and exit codes (see [output-formatting.md](output-formatting.md)) |
| `internal/justifiapi/` | Generated OpenAPI types (see [api-sync.md](api-sync.md)) |
| `internal/errors/`, `internal/testutil/` | Domain sentinels and shared test helpers |
| `api/openapi-spec/` | Vendored OpenAPI artifact and drift report ([api-sync.md](api-sync.md)) |
| `.goreleaser/` | Per-OS GoReleaser configs |
| `.github/workflows/` | CI, release, sync workflows |
| `specs/` | Specifications |

Why root layout rather than `cmd/justifi/`: this repo produces a single binary and never exports library packages. The `cmd/` indirection only earns its keep when a module also serves as a library.

### Command framework

The CLI uses **cobra** for command parsing. Each top-level resource group is a self-contained unit (its own file or files, registering itself with the root command). This keeps cognitive scope narrow when adding or modifying a single resource.

**Global flag surface** (set on the root command, available to every subcommand):

| Flag | Source spec |
|------|------|
| `--profile` | [config-and-auth.md](config-and-auth.md) |
| `--sub-account` | [config-and-auth.md](config-and-auth.md) |
| `--no-token-cache` | [config-and-auth.md](config-and-auth.md) |
| `--json` | [output-formatting.md](output-formatting.md) |
| `--output` | [output-formatting.md](output-formatting.md) |
| `--no-color` | [output-formatting.md](output-formatting.md) |
| `--verbose` / `-v` | [output-formatting.md](output-formatting.md) |
| `--quiet` / `-q` | [output-formatting.md](output-formatting.md) |
| `--timeout` | [http-client.md](http-client.md) |
| `--retries` | [http-client.md](http-client.md) |

**Help requirements** (enforced by code review, not lint, in v1):
- Every command and every flag has a non-empty short description.
- Every command has a long description with at least one example invocation.
- Examples use realistic JustiFi-prefix IDs (`py_…`, `sa_…`).

### Help and documentation

The cobra command tree is the single source for all help and reference documentation. Nothing in this list is hand-written or maintained separately — each output is generated from the same command, flag, and example definitions, so they cannot drift from the CLI's actual behavior.

| Output | Form | When produced |
|--------|------|---------------|
| Interactive help | Cobra's built-in nested `--help` at every level (`justifi --help`, `justifi payments --help`, `justifi payments list --help`) and `justifi help [command]` | Always available in the binary |
| Man pages | A man-page tree generated from the command tree | At build time; shipped in release artifacts (deb/rpm/Homebrew) |
| Shell completions | `justifi completion <bash\|zsh\|fish\|powershell>` subcommand | Generated on demand by the binary |
| Markdown reference | A markdown doc tree generated from the command tree, for a documentation site | By a `make docs` target and/or CI |

Interactive nested help is the baseline contract: an AI agent or human can learn any command — its flags, arguments, and an example — from `--help` alone, without reading source or external docs. Man pages, completions, and markdown are convenience surfaces over the same definitions.

### Guardrails

The repo must have working pre-commit lint+test hooks, pre-push integration-test hooks, and a conventional-commit message check **before any application source is committed**. Phase 1 of the foundation work list enforces this ordering.

- **Linter:** `golangci-lint` with a configuration adapted from `~/personal/go-blueprint`. Service-only linters (`rowserrcheck`, `sqlclosecheck`, blueprint-vet plugin, pgxkit/canonlog forbidigo rules) are removed. Generated code (`internal/justifiapi/`) is excluded.
- **Formatter:** `go fmt` is run as part of `make lint`.
- **Hooks:** `lefthook` runs `make lint` and `make test` pre-commit; runs `make test-all` pre-push; rejects non-conforming commit messages.
- **Commit messages:** Conventional Commits. Allowed types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `perf`, `ci`, `build`, `style`, `revert`. Scope is optional; when present, names a resource (`feat(payments): …`) or a layer (`fix(client): …`).

### Release model

The CLI is shipped as platform-native binaries via **GoReleaser**, with one config file per OS executed on a native GitHub Actions runner. Each platform handles its own toolchain (codesigning, notarization, package-manager publishing) without cross-compile workarounds.

| Platform | Outputs |
|----------|---------|
| Linux (amd64, arm64) | Tarballs, deb, rpm, multi-arch Docker image |
| macOS (amd64, arm64) | Tarballs; Homebrew tap update (after a tap exists) |
| Windows (amd64) | Zip; Winget manifest update (after a manifest repo exists) |

Version information (`Version`, `Commit`, `Date`) is injected via ldflags at build time and surfaced by a `justifi version` subcommand.

**Release trigger:** push a tag `vX.Y.Z` to the default branch. CI runs first (`canary-test.yml`); on green, three parallel GoReleaser jobs publish to the same GitHub Release. A failed platform job does not roll back; humans decide fix-forward vs yank.

**Local validation:** `make build-snapshot` runs GoReleaser in snapshot mode on the host OS to catch broken release configs before tagging.

### GitHub Actions workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `test.yml` | push, pull_request | Matrix CI (Go 1.26 × Linux/macOS/Windows). Runs `make ci`. Required for merge. |
| `canary-test.yml` | `workflow_call` (reusable) | Production-flag build + offline tests; sandbox e2e tests when secrets present. |
| `test-snapshot.yml` | push, PR touching `.goreleaser/**` | Per-OS `goreleaser --snapshot` to validate release configs. |
| `release.yml` | push tag `v*` | Calls `canary-test.yml`; on green, runs per-OS GoReleaser jobs. |
| `sync-openapi.yml` | scheduled (daily) + manual | Watches `justifi-tech/public-docs`, regenerates types, opens a PR. See [api-sync.md](api-sync.md). |
| `install-test.yml` | scheduled (hourly, once package channels exist) | Exercises each install channel; alerts on failure. Activated after the first stable release. |

The Linux/macOS/Windows matrix is the canonical "supported platforms" list. Adding a platform means adding it to both the CI matrix and the GoReleaser configs in the same change.

### Test layering

This is a thin CLI: most commands parse flags, make one API call, and render the result. There is little business logic in the command layer, so testing is concentrated where logic actually lives, not spread thinly across every command.

| Area | What is tested | How |
|------|----------------|-----|
| `internal/client` | Retry, idempotency, sub-account header, error translation, pagination | Direct, against `httptest.Server` (real round-trip) |
| `internal/output` | Table truncation, currency/timestamp formatting, JSON envelope shapes, exit-code mapping, TTY detection | Direct unit tests in the package |
| `internal/config` | Minimal — one test for credential resolution-order precedence and one for the token-cache validity window. Nothing exhaustive. | Direct unit tests in the package |
| api-sync scripts | Drift report, coverage check | Direct unit tests |
| Command layer | Only resource-specific validation/orchestration (mutual exclusions, required-flag errors, enum validation, file upload/download) | Direct tests of the validation functions; orchestration tests run the real command against the client wired to a stub HTTP transport |
| End-to-end | That a real API call returns the expected result | A small sandbox smoke suite of read-only calls |

**What is deliberately not tested:** happy-path passthrough verbs (list/get/create that just call the client and render). A mock-based test of those asserts only that the plumbing calls the mock — it verifies nothing and breaks on refactors. Passthrough is covered by the sandbox smoke suite.

**Mocking and interface rules** (see [http-client.md](http-client.md), [resource-command-template.md](resource-command-template.md)):
- **The HTTP client is concrete and has no interface.** It is faked at the `http.RoundTripper` transport seam — inject a `*http.Client` whose transport is a stub (or an `httptest.Server`) and run the real client. This is how both the client and command-orchestration tests work.
- **gomock + consumer-defined interfaces are reserved for non-HTTP behavioral seams** (interactive prompts, persistent config storage, OS actions). v1 has none of these, so v1 ships no generated mocks. When such a seam appears: the consumer declares the interface, the mock is generated next to it by gomock (never handwritten), and no test ever asserts a mock against the real implementation.
- Shared tools (`internal/output`, `internal/config`) are tested once in their own package; commands never re-test formatting or config behavior.

- `make test` runs only unit tests (`-short`). Pre-commit runs this.
- `make test-all` runs every test, including the sandbox smoke suite, when sandbox credentials are present in env. Pre-push runs this; absent credentials, the smoke suite skips with a notice rather than failing.

### Make target surface

The Makefile is the single command surface for all developer and CI workflows. The set of targets that must exist (purpose, not implementation):

| Target | Purpose |
|--------|---------|
| `help` | Print available targets |
| `setup` | Install dev tools and activate hooks |
| `lint` | Format + static analysis |
| `test` | Unit tests only |
| `test-all` | All tests including sandbox e2e |
| `ci` | What `test.yml` invokes |
| `build` | Host-platform binary for local iteration |
| `build-snapshot` | Snapshot GoReleaser build for the host OS |
| `run` | `go run .` passthrough |
| `generate` | Regenerate mocks and OpenAPI types ([api-sync.md](api-sync.md)) |
| `clean` | Remove build artifacts and Go caches |

Cross-compilation is not a Makefile concern; GoReleaser owns it.

## Notes

- **Why GoReleaser on native runners.** Cross-compiling Mac binaries from Linux requires cgo workarounds, mishandles codesigning/notarization, and breaks per-OS package manager publishing. Native runners cost a few extra minutes per release and eliminate that whole class of problems.
- **Why hand-roll commands instead of generating them.** Generated CLIs produce mechanical flag names, awkward help, and no human curation of what matters. See [http-client.md](http-client.md) for the broader trade-off.

## References

- `~/personal/go-blueprint/` — source of build, lint, hook, and test conventions.
- [Cobra](https://github.com/spf13/cobra)
- [Conventional Commits](https://www.conventionalcommits.org/)
- [lefthook](https://github.com/evilmartians/lefthook)
- [golangci-lint](https://golangci-lint.run/)
- [GoReleaser](https://goreleaser.com/)
- JustiFi API docs: https://docs.justifi.tech/
