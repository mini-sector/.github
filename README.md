# .github

Org-wide defaults and shared CI/CD building blocks for **mini-sector**: reusable
GitHub Actions workflows and composite actions used across the Rust backend
workspace, the Rust telemetry client, the TypeScript frontend, Terraform
infra, and the Argo CD gitops repo.

## Why the path has `.github/.github/workflows`

This repository is itself named `.github`, and reusable workflows must live
under `.github/workflows/` for GitHub to discover them. That means, relative
to the repo root, the workflows sit at `./.github/workflows/*.yml` on disk —
and when another repo references them, the reference is
`mini-sector/.github/.github/workflows/rust-ci.yml@main` (org **/** repo **/**
path-within-repo). The doubled segment is just the repo name (`.github`)
followed by the in-repo path (`.github/workflows/...`); it is not a mistake
and there is no third `.github` level to add. Do not "fix" it.

## Versioning policy

Callers currently reference workflows and actions at `@main`. Once the
contents stabilize, tag a `v1` release and switch all callers to `@v1` (and
subsequent `v2`, etc. on breaking changes).

## Required org setting

If this repo is ever made private, an org owner must enable access for other
org repositories under **Settings → Actions → General → Access**, or callers
in other repos won't be able to resolve these reusable workflows/actions.

## Composite actions

### `actions/setup-rust`

Installs a Rust toolchain and enables `Swatinem/rust-cache`.

| Input | Default | Description |
|---|---|---|
| `toolchain` | `stable` | Rust toolchain to install |
| `components` | `rustfmt, clippy` | Comma-separated rustup components |
| `cache-key` | `""` | Extra segment added to the rust-cache key |

### `actions/azure-login`

Logs in to Azure via OIDC federated credentials (no service principal
secrets). The calling workflow/job **must** set `permissions: id-token: write`.

| Input | Required | Description |
|---|---|---|
| `client-id` | yes | Azure AD application (client) ID |
| `tenant-id` | yes | Azure AD tenant ID |
| `subscription-id` | yes | Azure subscription ID |

### `actions/setup-terraform`

Installs Terraform, runs `terraform init`, and selects (or creates) a
workspace. Backend configuration comes from the calling repo's own backend
block / env vars — this action hardcodes none.

| Input | Default | Description |
|---|---|---|
| `terraform-version` | `~1.9` | Terraform version constraint |
| `workspace` | *(required)* | Terraform workspace name, e.g. `dev` or `prod` |
| `working-directory` | `.` | Directory containing the Terraform configuration |

## Reusable workflows

### `rust-ci.yml`

fmt, clippy, and test for a Rust crate or workspace, with an optional
`cargo-deny` check.

| Input | Type | Default | Description |
|---|---|---|---|
| `workspace` | boolean | `false` | Pass `--workspace` to cargo commands |
| `toolchain` | string | `stable` | Rust toolchain to install |
| `run-deny` | boolean | `false` | Run `cargo-deny check` after tests |

```yaml
# in mini-sector/platform/.github/workflows/ci.yml
name: CI
on:
  pull_request:
  push:
    branches: [main]
jobs:
  rust:
    uses: mini-sector/.github/.github/workflows/rust-ci.yml@main
    with:
      workspace: true
```

### `docker-build.yml`

Builds and pushes a service image, then bumps the image tag in the gitops
repo. It only pushes images and commits the tag bump — it never deploys or
touches a cluster; Argo CD reconciles the gitops repo separately.

| Input | Type | Default | Description |
|---|---|---|---|
| `service` | string | *(required)* | Crate/service name; also the image name |
| `dockerfile` | string | `Dockerfile` | Path to the Dockerfile |
| `context` | string | `.` | Docker build context |
| `registry` | string | `ghcr.io` | Container registry to push to |
| `gitops-repo` | string | *(required)* | Repo containing the Kustomize overlays, e.g. `mini-sector/gitops` |
| `gitops-path` | string | *(required)* | Directory containing the Kustomize overlay to update |
| `build-args` | string | `""` | Newline-separated `KEY=value` pairs passed to docker build |

| Secret | Required | Description |
|---|---|---|
| `gitops-token` | yes | PAT or GitHub App token with write access to the gitops repo |

```yaml
# in mini-sector/platform/.github/workflows/release.yml
name: Release
on:
  push:
    branches: [main]
jobs:
  gateway:
    uses: mini-sector/.github/.github/workflows/docker-build.yml@main
    with:
      service: gateway
      gitops-repo: mini-sector/gitops
      gitops-path: apps/gateway/overlays/dev
    secrets:
      gitops-token: ${{ secrets.GITOPS_TOKEN }}
```

### `terraform.yml`

fmt, validate, and plan (and optionally apply) Terraform against Azure via
OIDC.

| Input | Type | Default | Description |
|---|---|---|---|
| `workspace` | string | *(required)* | `dev` or `prod` |
| `working-directory` | string | `.` | Directory containing the Terraform configuration |
| `apply` | boolean | `false` | Apply the plan after it succeeds; otherwise stop after plan |

| Secret | Required | Description |
|---|---|---|
| `azure-client-id` | yes | Azure AD application (client) ID |
| `azure-tenant-id` | yes | Azure AD tenant ID |
| `azure-subscription-id` | yes | Azure subscription ID |

```yaml
# in mini-sector/infra/.github/workflows/terraform.yml
name: Terraform
on:
  pull_request:
  push:
    branches: [main]
jobs:
  plan:
    uses: mini-sector/.github/.github/workflows/terraform.yml@main
    with:
      workspace: dev
      apply: ${{ github.ref == 'refs/heads/main' }}
    secrets:
      azure-client-id: ${{ secrets.AZURE_CLIENT_ID }}
      azure-tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      azure-subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```
