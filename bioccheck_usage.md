# Rapid Feedback Container Check (`bioccheck.yml`)

The **Container Check** workflow runs `R CMD check` and `BiocCheck` inside the official `bioconductor/bioconductor_docker` container. Because system libraries and R dependencies are pre-installed in the container, this workflow runs rapidly and is ideal for PRs and continuous commits. It tests *only* on Linux and *only* against one version of Bioconductor that is determined dynamically based on the name of the branch that is being tested directly or by Pull Request. This is sufficient for most development purposes.

## Key Features
* **Dynamic Container Matching**:
  * Pull requests against `RELEASE_X_Y` branches automatically use `bioconductor/bioconductor_docker:RELEASE_X_Y`.
  * Pull requests or pushes to `devel`, `main`, `master`, or feature branches automatically use `bioconductor/bioconductor_docker:devel`.
* **R Dependency Caching**: Caches R package dependencies in `/usr/local/lib/R/site-library` keyed on `depends.Rds` and R version.
* **Integrated Checks**: Executes `R CMD check` (configurable error threshold via `error_on`) and `BiocCheck` (with `quit-with-status = TRUE`).
* **Optional Features**: Test coverage reporting via Codecov and `pkgdown` site build/deployment.

## Usage

We recommend using our interactive [Workflow Generator](https://lwaldron.github.io/workflows/) to build your `.github/workflows/bioccheck.yml` file. It allows you to easily toggle optional features like Codecov and `pkgdown` and outputs perfectly formatted YAML.

### Repository Setup for Optional Features

Even if you use the Workflow Generator, you must perform these one-time repository settings to use optional features:

**For Codecov:**
1. **Obtain Codecov Token**: Log in to [Codecov](https://about.codecov.io/), link your repository, and copy the repository upload token.
2. **Set Repository Secret**: In your GitHub repository, navigate to **Settings** &rarr; **Secrets and variables** &rarr; **Actions** &rarr; **New repository secret**. Create a secret named `CODECOV_TOKEN` containing your token.

**For `pkgdown`:**
1. **Configure GitHub Pages Source**: In your package repository on GitHub, navigate to **Settings** &rarr; **Pages**. Under **Build and deployment** &rarr; **Source**, select **GitHub Actions** (do NOT select "Deploy from a branch").

### Manual Configuration (Fallback)
If you prefer to configure the workflow manually, you can copy the following template and adjust the `with:` inputs, `secrets:`, and `permissions:` blocks according to your needs:

```yaml
name: CI

on:
  push:
    branches: [ devel, main, master, RELEASE_* ]
  pull_request:
    branches: [ devel, main, master, RELEASE_* ]

jobs:
  bioccheck:
    # Required only if you set enable_pkgdown: true
    permissions:
      pages: write
      id-token: write
    uses: lwaldron/workflows/.github/workflows/bioccheck.yml@v1
    with:
      error_on: 'warning'
      enable_pkgdown: false
    # Required only if you want test coverage via Codecov
    # secrets:
    #   CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
```

## Workflow Inputs & Parameters for `bioccheck.yml`

| Input / Secret | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `enable_pkgdown` | `boolean` | `false` | Enable building and deploying `pkgdown` site on release branch pushes. |
| `error_on` | `string` | `"warning"` | Error policy for `rcmdcheck` (`"never"`, `"note"`, `"warning"`, `"error"`). |
| `bioc_version` | `string` | `""` | Optional override for container tag (e.g. `devel`, `RELEASE_3_20`). |
| `secrets.CODECOV_TOKEN` | secret | `""` | Optional Codecov token for coverage uploads. |

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
