# 0004. In-place Package Installation for Dependencies

- **Status:** Accepted
- **Date:** 2026-07-24
- **Deciders:** Bioconductor Core Team, Levi Waldron

## Context and Problem Statement

Before running `R CMD check` or `BiocCheck`, all package dependencies must be installed in the CI environment. The legacy `waldronlab` workflow achieved this by calling `remotes::install_deps(dependencies = TRUE)`. This installs all packages listed in the `DESCRIPTION` file but does not actually build or install the *local package itself* into the container's R library.

Is it sufficient to only install dependencies (`install_deps`), or should the local package under test also be installed into the environment prior to checking?

## Decision

We will use `remotes::install_local(".", dependencies = TRUE)` instead of `remotes::install_deps()`.

## Alternatives Considered

- **`remotes::install_deps()`**: This was the previous behavior. It is faster because it only pulls down upstream dependencies. However, it was rejected because some downstream tests, vignettes, or dynamic checking logic might assume the package itself is installed and loadable via `library(pkgname)`. Relying on the `rcmdcheck` isolated build process can sometimes mask environment mismatches.
- **`devtools::install()`**: This is functionally similar to `remotes::install_local()`, but requires the heavier `devtools` suite as a dependency. `remotes` is lighter and already required for dependency resolution.

## Consequences

- **Higher Fidelity**: By explicitly installing the local package alongside its dependencies, we guarantee the container's R library accurately reflects the full runtime state of the package as it would be experienced by an end user.
- **Slightly Slower Setup**: Installing the local package takes a few extra seconds during the dependency installation step.
- **Immediate Failure on Install Errors**: If the package fails to compile or install at a fundamental level, the CI will fail immediately during the dependency installation step, rather than later during the checking steps, providing faster and clearer feedback for compilation-level issues.
