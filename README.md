# Bioconductor GitHub Actions Workflows

This repository (`lwaldron/workflows`) hosts reusable GitHub Actions workflows for R/Bioconductor package development.

---

## Overview: Two Complementary Testing Patterns

Two complementary workflows are available depending on your testing goals:

| Purpose | Workflow Reference | Target Environment | Managed By |
| :--- | :--- | :--- | :--- |
| **1. Rapid Feedback Check** | `lwaldron/workflows/.github/workflows/bioccheck.yml@v1` | Linux (`bioconductor_docker` container) | Bioconductor Core Team / Maintainers |
| **2. Comprehensive Pre-Release Check** | `r-universe-org/workflows/.github/workflows/build.yml@v3` | Multi-OS (Linux, macOS, Windows, WebAssembly) + `BiocCheck` | R-Universe Team |

---

## 1. Rapid Feedback Container Check (`bioccheck.yml`)

The **Container Check** workflow runs `R CMD check` and `BiocCheck` inside the official `bioconductor/bioconductor_docker` container. Because system libraries and R dependencies are pre-installed in the container, this workflow runs rapidly and is ideal for PRs and continuous commits. It tests *only* on Linux and *only* against one version of Bioconductor that is determined dynamically based on the name of the branch that is being tested directly or by Pull Request. This is sufficient for most development purposes.

### Key Features
* **Dynamic Container Matching**:
  * Pull requests against `RELEASE_X_Y` branches automatically use `bioconductor/bioconductor_docker:RELEASE_X_Y`.
  * Pull requests or pushes to `devel`, `main`, `master`, or feature branches automatically use `bioconductor/bioconductor_docker:devel`.
* **R Dependency Caching**: Caches R package dependencies in `/usr/local/lib/R/site-library` keyed on `depends.Rds` and R version.
* **Integrated Checks**: Executes `R CMD check` (configurable error threshold via `error_on`) and `BiocCheck` (with `quit-with-status = TRUE`).
* **Optional Features**: Test coverage reporting via Codecov and `pkgdown` site build/deployment.

### Example Usage (`.github/workflows/bioccheck.yml` in your package repo)

```yaml
name: CI

on:
  push:
    branches: [ devel, main, master, RELEASE_* ]
  pull_request:
    branches: [ devel, main, master, RELEASE_* ]

jobs:
  bioccheck:
    uses: lwaldron/workflows/.github/workflows/bioccheck.yml@v1
    with:
      enable_pkgdown: false
      error_on: 'warning'
    secrets:
      CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
```

When `enable_pkgdown: true`, the calling job must grant the permissions required for GitHub Pages deployment. The reusable workflow's `deploy` job declares `pages: write` and `id-token: write`, but these permissions must also be granted by the caller because GitHub Actions enforces the intersection of caller and called workflow permissions:

```yaml
jobs:
  bioccheck:
    permissions:
      pages: write
      id-token: write
    uses: lwaldron/workflows/.github/workflows/bioccheck.yml@v1
    with:
      enable_pkgdown: true
      error_on: 'warning'
    secrets:
      CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
```

---

## 2. Comprehensive Multi-Platform Check (`r-universe-org`)

For multi-platform verification across Linux, macOS, and Windows prior to release or submission, use the official **R-Universe workflow** (`r-universe-org/workflows/.github/workflows/build.yml@v3`).

When `organization: bioconductor` is provided, R-Universe automatically invokes `BiocCheck` via `r-universe-org/actions/bioc-check@v12`.

### Example Usage (`.github/workflows/runiverse.yml` in your package repo)

```yaml
name: R-Universe Build & BiocCheck

on:
  workflow_dispatch:           # Manual trigger before release/submission
  push:
    branches: [ devel ]        # Run on pushes to devel

jobs:
  build:
    uses: r-universe-org/workflows/.github/workflows/build.yml@v3
    with:
      universe: ${{ github.repository_owner }}
      organization: bioconductor
```

For additional configuration options and inputs, refer to the upstream [r-universe-org/workflows repository](https://github.com/r-universe-org/workflows).

---

## Workflow Inputs & Parameters for `bioccheck.yml`

| Input / Secret | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `enable_pkgdown` | `boolean` | `false` | Enable building and deploying `pkgdown` site on release branch pushes. |
| `enable_docker` | `boolean` | `false` | Enable Docker image build and push. |
| `error_on` | `string` | `"warning"` | Error policy for `rcmdcheck` (`"never"`, `"note"`, `"warning"`, `"error"`). |
| `bioc_version` | `string` | `""` | Optional override for container tag (e.g. `devel`, `RELEASE_3_20`). |
| `secrets.CODECOV_TOKEN` | secret | `""` | Optional Codecov token for coverage uploads. |

---

## Versioning Strategy (`v1` vs `main`)

When calling reusable GitHub Actions workflows, the portion after `@` can be any git reference (a tag, branch, or SHA):

- **`@v1` (Recommended for callers)**: Pointing callers to `@v1` relies on a floating major version tag or branch (e.g. `v1`). Maintainers update `v1` whenever backwards-compatible fixes or updates are published, protecting consuming repos from unexpected breaking changes in future major revisions (`v2`).
- **`@main`**: Points to the bleeding-edge default branch. Useful for development or internal repos that always want the latest changes immediately.
- **`@v1.0.0`**: Pins to an exact release tag for strict reproducibility.

### How to Maintain Version Tags / Branches

1. Keep `main` as the default development branch.
2. When releasing a major version, create a tag or branch named `v1` pointing to `main`:
   ```bash
   # Create or move the floating v1 tag
   git tag -fa v1 -m "Release v1"
   git push origin v1 --force
   ```
