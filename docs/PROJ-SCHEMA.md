# PROJ-SCHEMA — terraform-provider-readthedocs

## No Persistence Layer

This project has **no database / SQL schema**. It is a Go Terraform/OpenTofu provider that talks to the
hosted **Read the Docs REST API v3** — all durable state lives remotely at Read the Docs, plus the
Terraform/OpenTofu **state file** Terraform itself manages. This doc therefore covers the artifacts the
project *does* define: provider/resource/datasource schemas, environment/token structure, state
interaction, and the file formats it ships or consumes.

## Provider Schema (`internal/provider/provider.go`)

| Attribute | Type | Optional | Sensitive | Env fallback | Description |
|-----------|------|----------|-----------|--------------|-------------|
| `token` | string | yes | yes | `READTHEDOCS_TOKEN` | API token, sent as `Authorization: Token …` (60 req/min). Required — error if neither set. |
| `base_url` | string | yes | no | `READTHEDOCS_BASE_URL` | API v3 root, no trailing slash. Default `https://app.readthedocs.org/api/v3`; Business uses `https://app.readthedocs.com/api/v3`. |

Token precedence: provider block > env var. Client is built once in `Configure` and shared with all
resources/data sources via provider context.

## Resource Schemas (`internal/provider/`)

Common shape: string `id` (RTD ID) + `json` passthrough of the raw API object on most resources.

### readthedocs_project (`res_project.go`)

| Attribute | Type | Notes |
|-----------|------|-------|
| id, name, slug, repository_url, repository_type, homepage, language, programming_language, default_version, default_branch, privacy_level, external_builds_privacy_level, versioning_scheme, readthedocs_yaml_path, organization, docs_url, home_url, json | string | `privacy_level` values from RTD (`public`/`private`); `docs_url`/`home_url` computed server-side |
| external_builds_enabled, analytics_disabled | bool | |
| analytics_code | string | |
| teams, tags | list(string) | |

API: `POST/GET/PATCH /projects/` — **no official DELETE**, so destroy is a no-op/unreportable.

### readthedocs_version (`res_rest.go`)

| Attribute | Type | Notes |
|-----------|------|-------|
| id, project, slug, privacy_level, json | string | `project`/`slug` force replacement; `id`/`json` computed |
| active, hidden | bool | PATCH `/projects/{p}/versions/{v}/` (activate/hide only) |

### readthedocs_build (`res_rest.go`)

| Attribute | Type | Notes |
|-----------|------|-------|
| id, project, version, commit, state, json | string | `version` defaults to `latest` (replace-only); `commit`/`state`/`id` computed; `state` from RTD (`triggered`/`finished`…) |
| success | bool | computed |
| triggers | map(string) | write-only trigger metadata; POST `/projects/{p}/versions/{v}/builds/` |

### readthedocs_sync_versions (`res_rest.go`)

| Attribute | Type | Notes |
|-----------|------|-------|
| id, project | string | POST `/projects/{p}/sync-versions/` |
| triggers | map(string) | |

### readthedocs_redirect (`res_more.go`)

| Attribute | Type | Notes |
|-----------|------|-------|
| id, project, type, from_url, to_url, description, json | string | full CRUD on `/projects/{p}/redirects/` |
| http_status, position | int64 | defaults 302 / 0 |
| force, enabled | bool | defaults false / true |

### readthedocs_environment_variable (`res_more.go`)

| Attribute | Type | Notes |
|-----------|------|-------|
| id, project, name, value, json | string | `value` is **Required + Sensitive + force-replace** — holding secrets; POST/GET/DELETE only (no PATCH, so any change is a destroy/create) |
| public | bool | default false |

### readthedocs_subproject (`res_more.go`)

| Attribute | Type | Notes |
|-----------|------|-------|
| id, parent, child, alias, json | string | POST/GET/DELETE `/subprojects/`; `parent`/`child`/`alias` force replacement |

### readthedocs_sharing (`res_more.go`, Business only)

| Attribute | Type | Notes |
|-----------|------|-------|
| id, project, access_type, description, expires, url, json | string | `access_type` required + force-replace; POST/PATCH/DELETE `/sharing/` |
| password | string | optional, **Sensitive** |
| token | string | computed, **Sensitive** (share link token) |
| allow_all | bool | default true |
| versions | list(string) | |

## Data Sources (`ds.go`)

18 data sources, all sharing one generic `listModel` (only the relevant subset is populated per source):
`project`, `projects`, `version`, `versions`, `build`, `builds`, `redirects`, `environment_variables`,
`subprojects`, `translations`, `organization`, `organizations`, `organization_projects`,
`organization_teams`, `remote_organizations`, `remote_repositories`, `embed`, `superproject`.
List sources return `result_count` (int64) + `results_json` (raw API array as JSON string);
`embed` additionally exposes `content`/`fragment`.

## State Interaction

- **Terraform/OpenTofu state** is the only local persistence this project touches — owned by the CLI
  tool, not the provider code. All resource attributes above (including sensitive `token`,
  `environment_variable.value`, `sharing.password`, `sharing.token`) are persisted **in plaintext
  inside tfstate**; keep state files out of git (`.gitignore` covers `*.tfstate`, `*.tfstate.*`,
  `.terraform/`, `.terraform.lock.hcl`).
- No remote-state backend is configured by this repo; examples use local state.

## File Formats the Project Defines / Consumes

| File | Format | Role |
|------|--------|------|
| `terraform-registry-manifest.json` | JSON | Registry metadata if the provider is ever published |
| `.goreleaser.yml` | YAML | Release build matrix + checksum/sign config |
| `.github/workflows/ci.yml`, `release.yml` | YAML | CI + tag-driven goreleaser release |
| `gpg-pubkey.asc` | armored PGP | Release-signing public key (`2EABB783…E16D0`) |
| `~/.terraformrc` (user-side) | HCL | `dev_overrides` + `filesystem_mirror` pointing at `~/.local/share/terraform/plugins` — see README |
| `examples/github-utils/main.tf` | HCL | Example config for project `github-utils` (only tracked file in that dir) |
| `examples/github-utils/.terraform.lock.hcl` | HCL | Provider version lockfile, produced locally at `init`; **gitignored**, never tracked |

## Auth / Token Structure (structure only — never values)

- `READTHEDOCS_TOKEN` — RTD API token; sourced in the monorepo from `.envrc.dc`
  `secrets.readthedocs.token` (Infisical-managed); injected via direnv, never committed.
- `READTHEDOCS_BASE_URL` — optional endpoint override (.org vs .com Business tier).
