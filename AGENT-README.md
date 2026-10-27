# AGENT-README

> Orientation for AI coding agents working in this repo. Human docs: README.md; repo rules in CLAUDE.md.

## Type

terraform+go

## Purpose

From-scratch Terraform/OpenTofu provider for the Read the Docs API v3 — projects, versions, builds, redirects, environment variables, subprojects, sharing (source address `the-robot-lives/readthedocs`). Written directly against the official v3 docs; not a fork.

## Stack

Go provider; local compile install into `~/.local/share/terraform/plugins` (same layout as other `noizu/*` house providers).

## Dev mode

```bash
make compile && make test     # build binary + run tests
./scripts/build-provider.sh   # build + install into the local plugin dir (both registry.terraform.io and registry.opentofu.org layouts)
```

## Ports / URLs

- none (library/CLI)

## Tests

```bash
make test    # go test ./...
```

## Gotchas

- No registry listing — wire up via `~/.terraformrc` `dev_overrides` (`the-robot-lives/readthedocs` → `~/.local/share/terraform/plugins`) and/or `filesystem_mirror`.
- Auth: `READTHEDOCS_TOKEN` (or provider `token`); Business deployments use `base_url = "https://app.readthedocs.com/api/v3"`; rate limit is 60 req/min authenticated.
- API v3 gaps are by design: no project DELETE, no custom domains, no webhooks — don't add resources for surfaces the public API lacks (README § "Not implementable").
