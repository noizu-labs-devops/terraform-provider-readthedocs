# PROJ-ARCH — terraform-provider-readthedocs

<!-- 〚📁:ec51fbcd-2e90-5920-ae4c-510bb8bb3f4f〛 PROJ-ARCH :: Architecture — CLI plugin model, provider/rtdapi split, REST data flow, delivery, key decisions -->

## Overview

A from-scratch **Terraform/OpenTofu provider** for the hosted **Read the Docs REST API v3**, written in
Go on `terraform-plugin-framework`. It lets portfolio docs infrastructure (e.g. the `github-utils`
project) be declared in HCL: projects, versions, builds (push trigger), sync-versions, redirects,
environment variables, subprojects, and Business-tier sharing can be created/read/updated, plus 18
read-only data sources.

It is deliberately **not** a fork of `BarnabyShearer/readthedocs` (abandoned 2022, projects only) or of
any MCP — client and schemas are written from the official v3 API docs. Provider type name is
`readthedocs`, `Metadata().Version` is `0.2.0`. Distribution is house-style: a local build installed
into the shared plugin mirror at `~/.local/share/terraform/plugins` (same pattern as SigNoz /
`noizu/foryou` providers); GitHub `v*` releases via goreleaser exist for checksum/signing but there is
no registry listing (the tracked `terraform-registry-manifest.json` only declares protocol version 6.0).

## System Diagram

```mermaid
graph LR
    subgraph User machine
        TF[tofu / terraform CLI] -->|gRPC plugin protocol| P[provider binary]
        TF --- ST[(tfstate)]
    end
    subgraph This repo
        P --> C[rtdapi.Client]
        P --- S[resource + datasource schemas]
    end
    C -->|HTTPS, Authorization: Token| RTD[(Read the Docs API v3)]
```

## Core Components

| Component | Purpose |
|-----------|---------|
| `main.go` | Entry point — serves the provider over the plugin handshake |
| `internal/provider/provider.go` | Provider config (`token`, `base_url`), builds shared `rtdapi.Client`, registers 8 resources + 18 data sources |
| `internal/provider/res_project.go` | `readthedocs_project` (the only full CRUD-ish resource) |
| `internal/provider/res_rest.go` | `version`, `build`, `sync_versions` resources |
| `internal/provider/res_more.go` | `redirect`, `environment_variable`, `subproject`, `sharing` resources |
| `internal/provider/ds.go` | All data sources via one **generic datasource** (`genericDS` + shared `listModel`) parameterized by name/read-fn |
| `internal/provider/helpers.go` | Shared JSON→state mapping helpers |
| `internal/rtdapi/client.go` | From-scratch REST client: auth header, endpoint builders, RTD JSON shapes (+ unit tests) |

→ *Components ↔ directories: see [PROJ-LAYOUT.md](PROJ-LAYOUT.md)*

## Data Flow

`Configure` resolves token (provider block > `READTHEDOCS_TOKEN`) and base URL
(provider block > `READTHEDOCS_BASE_URL` > `.org` default) and stores one `*rtdapi.Client` in provider
context. Every resource/datasource operation is then a stateless REST call: plan/apply diffs local
schema state vs remote GET; writes map to the narrow verbs RTD v3 allows (notably: **no project
DELETE**, env-vars have no PATCH, version changes are only activate/hide via PATCH). Every resource
keeps a `json` attribute with the raw API object as an escape hatch; list data sources return
`result_count` + `results_json`.

The client is a thin `net/http` wrapper: one `http.Client` with a 45s timeout, `Authorization: Token …`
on every request, JSON bodies, paginated collections walked by `limit`/`offset` with a hard stop past
10,000 accumulated results, error bodies truncated to 500 bytes, and **no retry/backoff** — an RTD 429
surfaces as a hard provider error.

## Infrastructure / Delivery

- **Local install (house path)**: `scripts/build-provider.sh` → `go build` into
  `~/.local/share/terraform/plugins` for both `registry.opentofu.org` and `registry.terraform.io`
  mirror layouts; `~/.terraformrc` `dev_overrides` skips `init` downloads.
- **CI** (`.github/workflows/ci.yml`): `go test ./...` + `go build` on pushes to `main`/`stage` and on
  every pull request, plus a warn-only `ci-timing` job that reports PR step/run durations against
  `PR_STEP_MAX_MINUTES`/`PR_RUN_MAX_MINUTES` variables. **Pushes to `develop` are not built** — develop
  is only exercised through PR runs. CI pins `go-version: "1.22"` while `go.mod` declares `go 1.25.0`
  and `.tool-versions` pins golang 1.25.8; the test/build steps therefore run on the older default
  toolchain unless the module's `go` directive forces a download.
- **Release** (`.github/workflows/release.yml`): goreleaser on `v*` tags (multi-arch, checksums,
  GPG-signed with `gpg-pubkey.asc`, key imported via `crazy-max/ghaction-import-gpg`), `contents: write`.
- **No servers, queues, or DB** — the provider is a short-lived child process of the CLI.

## Key Decisions

- **From-scratch client** instead of forking the abandoned community provider: full v3 coverage
  (orgs, remote VCS, embed, sharing) with schemas we control.
- **Generic datasource pattern**: 18 read-only sources share one model/read plumbing, cutting
  boilerplate; per-source behavior is a name + read function + optional extra attributes.
- **`json` passthrough attributes**: RTD API evolves faster than the provider; raw JSON preserves
  access to undocumented fields.
- **Local mirror over registry**: no HashiCorp Registry listing needed; consistent with the house
  SigNoz/foryou provider installs.
- **API-limit honesty**: resources intentionally omit operations the public API does not offer
  (documented in README rather than faked).

## Notes

- Sensitive values (`token`, env-var values, sharing passwords) live in plaintext tfstate — see
  [PROJ-SCHEMA.md](PROJ-SCHEMA.md) § State Interaction.
