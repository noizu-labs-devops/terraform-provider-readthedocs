# Project Layout — terraform-provider-readthedocs

From-scratch Terraform/OpenTofu provider for the **Read the Docs API v3** (`the-robot-lives/readthedocs`).
Go + terraform-plugin-framework. ~2.2k LOC; single-level tree, no `layout/` extraction needed.

```text
terraform-provider-readthedocs/
├── main.go                        # Entry point: plugin serve (tf6server-style handshake via plugin-framework)
├── internal/
│   ├── provider/                  # Provider + all resource/datasource schemas → see below
│   │   ├── provider.go            # Provider def `readthedocs` v0.2.0: token/base_url config, registries
│   │   ├── res_project.go         # readthedocs_project resource (CRUD on /projects/, no official DELETE)
│   │   ├── res_rest.go            # version, build (push/rebuild), sync_versions resources
│   │   ├── res_more.go            # redirect, environment_variable, subproject, sharing (Business) resources
│   │   ├── ds.go                  # All 18 data sources (project(s), version(s), build(s), org(s), remote, embed, …)
│   │   └── helpers.go             # Shared JSON/type helpers for schemas
│   └── rtdapi/
│       ├── client.go              # From-scratch REST client for RTD API v3 (auth, endpoints, JSON shapes)
│       └── client_test.go         # Unit tests for the client
├── examples/
│   └── github-utils/              # Working example config (main.tf only; tfstate/lockfile are gitignored, not tracked)
├── scripts/
│   └── build-provider.sh          # House build: go build + install into local plugin mirror dirs
├── .github/
│   └── workflows/
│       ├── ci.yml                 # CI: go test + go build on push to main/stage and all PRs,
│       │                          #   plus a warn-only `ci-timing` PR job (step-sum threshold, never blocks)
│       └── release.yml            # goreleaser release on v* tags
├── .goreleaser.yml                # Release config (multi-OS/arch binaries, checksums)
├── gpg-pubkey.asc                 # GPG public key for release signing (2EABB783…E16D0)
├── terraform-registry-manifest.json  # Registry metadata (only needed if ever published)
├── go.mod / go.sum                # Go module (terraform-plugin-framework deps)
├── .tool-versions                 # Toolchain pin (golang 1.25.8) — asdf/mise
├── Makefile                       # compile / test / install helpers
├── README.md                      # Start here: auth, resource/datasource tables, local install path
├── CLAUDE.md                      # Claude Code guidance: commands + monorepo rules pointer
├── AGENT.md / AGENTS.md           # Multi-agent build rules (kept aligned with CLAUDE.md)
├── docs/                          # PROJ-LAYOUT/SCHEMA/ARCH/THREAT-MODEL + .summary twins
├── merge-notes.md                 # sep-1 branch-sweep notes (2026-09-01)
├── TODO.md                        # Remaining follow-ups
├── LICENSE                        # MIT
└── .gitignore                     # Build outputs, .terraform, tfstate, .claude/worktrees/
```

## Key Files Requiring Setup

| File | Action |
|------|--------|
| `READTHEDOCS_TOKEN` (env) | Required — API token, fed from monorepo `.envrc.dc` `secrets.readthedocs.token`; never commit |
| `~/.terraformrc` | Optional — `dev_overrides` + `filesystem_mirror` pointing at `~/.local/share/terraform/plugins` (see README) |
| `examples/github-utils/main.tf` | Example config; do not `apply` without a token |

## Notes

- Docs set created 2026-09-01; THREAT-MODEL added 2026-10-07; swept 2026-10-27.
- `examples/github-utils/terraform.tfstate*` and `.terraform.lock.hcl` are local artifacts at apply time —
  transient, gitignored, never commit values. The tracked example is `main.tf` only.
- The canonical checkout is `Projects/terraform-provider-readthedocs` in the monorepo (gitlink). The
  former `Portfolio/Utilities/source/terraform-provider-readthedocs/` scaffolding tree no longer exists.
- Toolchain drift (documented, not changed by this doc): `.tool-versions` pins golang 1.25.8, `go.mod`
  declares `go 1.25.0`, while `ci.yml` sets `go-version: "1.22"`.
- `.claude/worktrees/` is gitignored canonical worktree placement (see repo CLAUDE.md).
