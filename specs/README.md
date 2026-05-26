# justifi-cli Specifications

> JustiFi API CLI

## How Specs Work

Specs are **steering documents** — they define WHAT to build and WHY, not HOW to implement.

**Workflow:**

1. **Spec phase** — We work through a spec until it's right
2. **Loop phase** — `loop.sh` runs agents that implement the spec

**Agents have autonomy** on implementation. The spec steers direction, the agent decides the code.

**Status transitions.** Humans move specs from Draft → Ready. The `/specd:audit` command manages Ready ↔ Implemented transitions — promoting clean specs to Implemented, demoting specs with new findings back to Ready.

**Future items:** Items marked with `(future)` are for reference only. Do not implement them — they belong to a later phase or another spec.

**Dependencies:** If a feature depends on another spec, check that spec's status. Only implement if the dependency is Ready or Implemented. Mark blocked features with "(blocked: specname)".

**Cross-references:** When referencing another spec in the body (Out of scope, Dependencies, inline text), use a real markdown link with the correct relative path.

**Work items** live in [specd_work_list.md](../specd_work_list.md). The `/specd:audit` command generates work items directly in specd_work_list.md based on gaps between specs and code. Humans and planning agents can also write directly to specd_work_list.md during spec phase.

## Status Legend

| Status      | Meaning                                                |
| ----------- | ------------------------------------------------------ |
| Draft       | Being specified — not ready for implementation         |
| Ready       | Spec complete, ready for implementation                |
| Implemented | Fully implemented                                      |
| Deprecated  | Superseded by another spec — kept for legacy reference |

---

## Foundation

| Spec | Status | Description |
|------|--------|-------------|
| [foundation](foundation.md) | Ready | Repo layout (root-dir Go), cobra command framework, Makefile, lefthook, golangci-lint, GitHub Actions, GoReleaser |
| [config-and-auth](config-and-auth.md) | Ready | Credential resolution (env + config file), profiles, sub-account scoping, OAuth token caching, `justifi auth ...` subcommands |
| [output-formatting](output-formatting.md) | Ready | JSON/table auto-selection by TTY, output envelopes, exit codes, pagination output |
| [http-client](http-client.md) | Ready | Bespoke `net/http` client behind an `internal/client.Client` interface (no SDK), retries, idempotency, error translation |
| [api-sync](api-sync.md) | Ready | Daily workflow bundling `public-docs` OpenAPI, regenerating types, drift report, auto-PR |
| [resource-command-template](resource-command-template.md) | Ready | Common contract every `justifi <resource> <verb>` command follows; referenced by all resource specs |

## Resources

| Spec | Status | Description |
|------|--------|-------------|
| [payments](payments.md) | Ready | `justifi payments` — list, get, create, capture, void |
| [payment-methods](payment-methods.md) | Ready | `justifi payment-methods` — tokenize and manage cards/bank accounts |
| [payment-method-groups](payment-method-groups.md) | Ready | `justifi payment-method-groups` — group payment methods |
| [refunds](refunds.md) | Ready | `justifi refunds` — full/partial refunds |
| [disputes](disputes.md) | Ready | `justifi disputes` — chargeback lifecycle and evidence |
| [payouts](payouts.md) | Ready | `justifi payouts` — settlement batches |
| [payout-holds](payout-holds.md) | Ready | `justifi payout-holds` — risk/compliance freezes (read-only) |
| [balance-transactions](balance-transactions.md) | Ready | `justifi balance-transactions` — ledger entries (read-only) |
| [ach-return-fees](ach-return-fees.md) | Ready | `justifi ach-return-fees` — returned-ACH charges (get only) |
| [checkouts](checkouts.md) | Ready | `justifi checkouts` — hosted/embedded payment sessions |
| [sub-accounts](sub-accounts.md) | Ready | `justifi sub-accounts` — platform merchant accounts |
| [fee-configurations](fee-configurations.md) | Ready | `justifi fee-configurations` — per-sub-account fee rates |
| [proceeds](proceeds.md) | Ready | `justifi proceeds` — platform fee share (read-only) |
| [reports](reports.md) | Ready | `justifi reports` — async CSV exports |
| [entities](entities.md) | Ready | `justifi entities <business\|identity\|address\|document\|bank-accounts\|terms-and-conditions\|provisioning>` — onboarding building blocks |
| [terminals](terminals.md) | Ready | `justifi terminals` — card-present terminal lifecycle |
| [terminal-orders](terminal-orders.md) | Ready | `justifi terminal-orders` — ordering physical terminals |

## Future

| Spec | Status | Description |
|------|--------|-------------|

<!-- Add your future/planned specs here -->
