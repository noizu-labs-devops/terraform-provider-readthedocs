# PROJ-LAYOUT.summary — terraform-provider-readthedocs

```text
terraform-provider-readthedocs/
├── main.go                    # Plugin entry point
├── internal/
│   ├── provider/              # Provider + 8 resources + 18 data sources
│   └── rtdapi/                # From-scratch RTD API v3 REST client (+ tests)
├── examples/github-utils/     # Example terraform config (main.tf; state/lock gitignored)
├── scripts/                   # build-provider.sh (local install)
├── .github/workflows/         # ci.yml (test+build on main/stage pushes + PRs, timing tracker),
│                              # release.yml (goreleaser on v* tags)
├── .goreleaser.yml            # Release config
├── gpg-pubkey.asc             # Release signing key
├── terraform-registry-manifest.json
├── go.mod / go.sum
├── .tool-versions
├── Makefile
├── CLAUDE.md / AGENT.md / AGENTS.md
├── README.md / merge-notes.md / TODO.md / LICENSE
└── docs/                      # PROJ-LAYOUT, PROJ-SCHEMA, PROJ-ARCH, THREAT-MODEL (+ .summary twins)
```
