# Rapid Feedback Container Check (`bioconductor/bioccheck` workflow)

The **Container Check** workflow runs `R CMD check` and `BiocCheck` inside the official `bioconductor/bioconductor_docker` Linux container. Because system libraries and R dependencies are pre-installed in the container, this workflow runs rapidly and is ideal for PRs and continuous commits. It tests *only* on Linux and *only* against one version of Bioconductor that is determined dynamically from the name of the branch that is being tested directly or by Pull Request. This is sufficient for most development and many Bioconductor package maintainers.

## Key Features
* **Dynamic Container Matching**:
  * Pull requests against `RELEASE_X_Y` branches automatically use `bioconductor/bioconductor_docker:RELEASE_X_Y`.
  * Pull requests or pushes to `devel`, `main`, `master`, or feature branches automatically use `bioconductor/bioconductor_docker:devel`.
* **R Dependency Caching**: Caches R package dependencies in `/usr/local/lib/R/site-library` keyed on `depends.Rds` and R version.
* **Integrated Checks**: Executes `R CMD check` (configurable error threshold via `error_on`) and `BiocCheck` (with `quit-with-status = TRUE`).
* **Optional Features**: Test coverage reporting via Codecov and `pkgdown` site build/deployment.

## Usage

### Basic Workflow

To just run `R CMD check` and `BiocCheck` on commits and PRs, you can copy the [ci.yml](ci.yml) template into `.github/workflows/ci.yml` and no additional configuration is needed. 

To add additional features like `COVR` and `pkgdown`, or change error sensitivity (`error/warning/note/never`), you can edit the above file, or use our interactive [Workflow Generator](https://bioconductor.github.io/workflows/).

### Repository Configuration for Optional Features

Even if you use the Workflow Generator, you must perform these one-time repository settings to use optional features:

**For Codecov:**
1. **Obtain Codecov Token**: Log in to [Codecov](https://about.codecov.io/), link your repository, and copy the repository upload token.
2. **Set Repository Secret**: In your GitHub repository, navigate to **Settings** &rarr; **Secrets and variables** &rarr; **Actions** &rarr; **New repository secret**. Create a secret named `CODECOV_TOKEN` containing your token.

**For `pkgdown`:**
1. **Configure GitHub Pages Source**: In your package repository on GitHub, navigate to **Settings** &rarr; **Pages**. Under **Build and deployment** &rarr; **Source**, select **GitHub Actions** (do NOT select "Deploy from a branch").

## Workflow Inputs & Parameters

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
