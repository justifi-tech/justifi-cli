# Config and Auth

| | |
|--------|----------------------------------------------|
| Status | Ready |
| Last Updated | 2026-05-26 |

## Overview

How the CLI obtains JustiFi credentials, switches between environments, scopes calls to a sub-account, and caches OAuth tokens. The CLI supports JustiFi's OAuth 2.0 client-credentials flow against `https://api.justifi.ai/oauth/token`, with credentials sourced from environment variables and/or a config file. Multiple named profiles are supported (e.g. `prod`, `sandbox`, `customer_X`).

**Scope:**
- Resolving credentials from env vars, config file, and command-line flags
- Named profiles in the config file
- Sub-account scoping (global flag + env + per-command override)
- OAuth token caching across CLI invocations
- Where the config file lives and its format

**Out of scope:**
- The HTTP request loop — see [http-client.md](http-client.md)
- Output formatting — see [output-formatting.md](output-formatting.md)
- Per-resource behavior — see resource specs

**Dependencies:** [foundation.md](foundation.md) for the cobra root and global flags.

**Design principles:**
- **Env vars win.** Twelve-factor; AI-tool friendly; trivial to script. The config file is a convenience for humans switching between accounts, not a primary mechanism.
- **No silent credential pickup from the home dir when env vars are set.** If `JUSTIFI_CLIENT_ID` is set, use it. Don't merge it with a config-file profile and produce surprising behavior.
- **Token cache is opt-out, never opt-in.** A fresh OAuth token every command would be wasteful and slow; we cache by default with a safe TTL margin.
- **No keychain in v1.** Adds OS-specific code paths and friction for AI tools / CI. Revisit later.

## Specification

### Credential resolution order

For each invocation, the CLI resolves the active credentials in this strict order. The first source that fully populates `client_id` and `secret_token` wins; the CLI does not merge across sources.

1. **Command-line flags** (highest precedence):
   - `--client-id <id>` and `--client-secret <secret>` — both must be present; if only one is set, error and exit `2` (usage error).
2. **Environment variables:**
   - `JUSTIFI_CLIENT_ID` and `JUSTIFI_SECRET_TOKEN` — both must be present.
3. **Config file profile:**
   - `--profile <name>` flag selects a named profile; if absent, `JUSTIFI_PROFILE` env var is consulted; if that is also absent, the profile named `default` is used.
   - The profile must define both `client_id` and `secret_token`. Missing fields → exit `2` with a message naming the file and profile.

If no source produces credentials, the CLI exits with code `2` and a message: `no JustiFi credentials found; set JUSTIFI_CLIENT_ID and JUSTIFI_SECRET_TOKEN or configure a profile in <config-path>`.

### Config file location and format

The config file is **TOML** at one of these paths (first existing wins):

1. `$JUSTIFI_CONFIG` if set (explicit override; if the file does not exist, exit `2`)
2. `$XDG_CONFIG_HOME/justifi/config.toml`
3. `$HOME/.config/justifi/config.toml`
4. `$HOME/.justifi/config.toml` (legacy/convenience fallback)

On Windows, `%APPDATA%\justifi\config.toml` is checked before the `$HOME/.config` paths.

**Format:**

```toml
default_profile = "sandbox"   # optional; defaults to "default" when absent

[profiles.default]
client_id     = "abc..."
secret_token  = "def..."
api_base      = "https://api.justifi.ai"   # optional override; defaults to https://api.justifi.ai
sub_account   = "sa_default..."             # optional; sets a per-profile default sub-account

[profiles.sandbox]
client_id     = "test_abc..."
secret_token  = "test_def..."
api_base      = "https://api.justifi.ai"
sub_account   = ""

[profiles.customer_alpha]
client_id     = "..."
secret_token  = "..."
sub_account   = "sa_alpha..."
```

**Recognized profile fields:** `client_id`, `secret_token`, `api_base`, `sub_account`. Unknown fields produce a warning on stderr but do not fail the command (forward compatibility).

**File permissions:** The CLI must refuse to read a config file with permissions broader than `0600` on Unix. On a permission violation, exit `2` with a remediation hint (`chmod 600 <path>`). On Windows, file ACL checks are skipped.

### Sub-account scoping

Many JustiFi endpoints accept or require a `Sub-Account` header that scopes the call to a specific merchant. Resolution order, first non-empty wins:

1. `--sub-account <id>` command flag (per-invocation override)
2. `JUSTIFI_SUB_ACCOUNT` env var
3. The active profile's `sub_account` field
4. Empty — call is made without a `Sub-Account` header

The literal value `""` (empty string) passed via `--sub-account ""` explicitly clears any profile or env default for that invocation.

### OAuth token caching

OAuth client-credentials tokens last approximately 24 hours per JustiFi. The CLI caches them locally to avoid a token-exchange on every command.

**Cache location:** `$XDG_CACHE_HOME/justifi/tokens.json` (falls back to `$HOME/.cache/justifi/tokens.json`, or `%LOCALAPPDATA%\justifi\tokens.json` on Windows). The directory is created with `0700`, the file with `0600`.

**Cache schema (per cached entry):**

```json
{
  "<sha256-of-client_id+api_base>": {
    "access_token": "...",
    "expires_at":   "2026-05-27T12:34:56Z",
    "fetched_at":   "2026-05-26T12:34:56Z",
    "api_base":     "https://api.justifi.ai"
  }
}
```

- The cache key is the SHA-256 hash of `client_id|api_base` so multiple profiles coexist and so the cache file does not contain `client_id` in cleartext.
- A token is considered valid if `now < expires_at - 60s` (60-second safety margin).
- On any error reading or parsing the cache file, the CLI proceeds as if the cache were empty and overwrites it on the next successful token fetch.
- The token cache file must be `0600`; if the existing file has broader permissions, the CLI rewrites it with `0600` on next write.

**Flags affecting caching:**
- `--no-token-cache` — bypass cache for this invocation; fetch a fresh token. Does not overwrite the cache.
- `justifi auth logout` subcommand — removes the entry for the current active credentials from the cache. `justifi auth logout --all` removes every entry.

### Auth subcommands

A small `auth` command group helps humans verify and manage credentials.

| Command | Purpose |
|---------|---------|
| `justifi auth whoami` | Print the resolved `client_id` (last 4 chars only), `api_base`, active profile name, and sub-account, plus whether a cached token is valid |
| `justifi auth login` | Fetch and cache a fresh token, then print the same fields as `whoami`. Useful as a connectivity check |
| `justifi auth logout` | Remove the cached token entry for the active credentials |
| `justifi auth logout --all` | Remove every cached token entry |
| `justifi auth config init` | Create `$XDG_CONFIG_HOME/justifi/config.toml` with a stub `[profiles.default]` section, chmodded `0600` |
| `justifi auth config path` | Print the resolved config-file path the CLI is using (or would use) |

`auth whoami` and `auth login` exit `0` on success, `1` on credential failure (auth rejected by JustiFi), `2` on misconfiguration (no creds at all).

### Behavior under non-TTY (AI tool) invocation

When stdout is not a TTY:
- `auth whoami` and `auth login` emit JSON instead of human text (see [output-formatting.md](output-formatting.md)).
- `auth config init` still creates the file but emits its path as JSON: `{"path": "..."}`.
- No interactive prompts are ever issued by any command in any code path. If a value is needed and not supplied, the CLI errors out.

### Environment variable summary

| Variable | Purpose |
|----------|---------|
| `JUSTIFI_CLIENT_ID` | OAuth client id |
| `JUSTIFI_SECRET_TOKEN` | OAuth client secret |
| `JUSTIFI_PROFILE` | Default profile name when `--profile` is absent |
| `JUSTIFI_SUB_ACCOUNT` | Default sub-account id when `--sub-account` is absent |
| `JUSTIFI_API_BASE` | Override API base URL (defaults to `https://api.justifi.ai`); useful for staging or local fakes |
| `JUSTIFI_CONFIG` | Override config-file path |

### Error messages

All credential-related errors must:
- Go to stderr.
- State which source the CLI was reading from (env, config file path + profile name, or flags).
- Suggest a remediation step.
- Exit with code `2` (usage error) when credentials are missing or malformed; `1` when JustiFi rejects them at the auth endpoint.

Example:
```
error: no JustiFi credentials found
  tried: JUSTIFI_CLIENT_ID env var, profile "default" in /Users/x/.config/justifi/config.toml
  hint: export JUSTIFI_CLIENT_ID and JUSTIFI_SECRET_TOKEN, or run `justifi auth config init`
```

## Notes

- **Why TOML, not YAML or JSON.** TOML is more forgiving than JSON for humans, less footgun-prone than YAML, and has a single canonical Go parser.
- **Why hash the cache key.** Defense in depth — a leaked token cache file should not directly expose a usable `client_id`. The token itself is sensitive but already short-lived.
- **Why no keychain in v1.** Adds OS-specific dependencies (Keychain on macOS, Secret Service on Linux, CredMan on Windows), complicates CI, and breaks headless AI-tool use unless we add fallbacks. The `0600` file with safe path is good enough for v1.
- **Why no interactive prompts.** AI tools and CI cannot answer prompts. We must fail fast and explicitly instead of hanging.

## References

- JustiFi Authentication docs: https://docs.justifi.tech/gettingStarted
- JustiFi token endpoint: `POST https://api.justifi.ai/oauth/token`
- [TOML](https://toml.io/)
- [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/)
