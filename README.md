# Bioconductor GitHub Actions Workflows

This repository (`Bioconductor/.github`) hosts reusable GitHub Actions workflows for R/Bioconductor package development. 

---

## Overview: Two Complementary Testing Patterns

Two complementary workflows are available depending on your testing goals:

| Purpose | Workflow Reference | Target Environment | Managed By |
| :--- | :--- | :--- | :--- |
| **1. Rapid Feedback Check** | `Bioconductor/.github/.github/workflows/bioccheck.yml@main` | Linux (`bioconductor_docker` container) | Bioconductor Core Team |
| **2. Comprehensive Pre-Release Check** | `r-universe-org/workflows/.github/workflows/build.yml@v3` | Multi-OS (Linux, macOS, Windows, WebAssembly) + `BiocCheck` | R-Universe Team |

---

## 1. Rapid Feedback Container Check (`bioccheck.yml`)

The **Container Check** workflow runs `R CMD check` and `BiocCheck` inside the official `bioconductor/bioconductor_docker` container. Because system libraries and R dependencies are pre-installed in the container, this workflow runs rapidly and is ideal for PRs and continuous commits. It tests _only_ on Linux and _only_ against one version of Bioconductor that is determined dynamically based on the name of the branch that is being tested directly or by Pull Request. This is sufficient for most development purposes.

### Key Features
* **Dynamic Container Matching**:
  * Pull requests against `RELEASE_X_Y` branches automatically use `bioconductor/bioconductor_docker:RELEASE_X_Y`.
  * Pull requests or pushes to `devel`, `main`, `master`, or feature branches automatically use `bioconductor/bioconductor_docker:devel`.
* **R Dependency Caching**: Caches R package dependencies in `/usr/local/lib/R/site-library` keyed on `depends.Rds` and R version.
* **Integrated Checks**: Executes `R CMD check` (failing on warnings) and `BiocCheck` (with `quit-with-status = TRUE`).
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
  check:
    uses: Bioconductor/.github/.github/workflows/bioccheck.yml@main
    with:
      enable_pkgdown: false
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
