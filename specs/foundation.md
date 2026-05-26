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

The HTTP-service layer from the blueprint is removed. Three test layers remain:

| Layer | Style | Network | Mocks |
|-------|-------|---------|-------|
| `internal/client` | Unit | `httptest.Server` | None — real HTTP loop |
| Resource command files | Unit | None | `MockClient` against the `internal/client` interface |
| End-to-end | Integration | Live JustiFi sandbox | None |

- `make test` runs only unit tests (`-short`). Pre-commit runs this.
- `make test-all` runs every test, including e2e, when sandbox credentials are present in env. Pre-push runs this; absent credentials, e2e tests skip with a notice rather than failing.
- Mocks are produced by `mockgen` in the same package as the consumer of the mocked interface (see [http-client.md](http-client.md) for details).

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
