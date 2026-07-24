# 0001. Use of Dynamic Matrix Container Tagging

- **Status:** Accepted
- **Date:** 2026-07-24
- **Deciders:** Bioconductor Core Team, Levi Waldron

## Context and Problem Statement

When testing Bioconductor packages via continuous integration, it is critical that the environment closely matches the target Bioconductor release. Upstream workflows often hardcoded the testing container (e.g., `ghcr.io/bioconductor/bioconductor_docker:devel`), meaning that even if a developer pushed a bugfix to a `RELEASE_X_Y` branch, the CI would test the code against the `devel` version of R and Bioconductor packages. This leads to false positives (tests failing due to newer dependencies) or false negatives (tests passing but failing in the actual release environment). 

How do we ensure that package tests run in the correct Bioconductor container version without requiring the package maintainer to manually specify it?

## Decision

We will dynamically resolve the container tag during a `set-matrix` setup job based on the Git ref name.

Specifically:
- If a pull request or push targets a branch prefixed with `RELEASE_`, the workflow will use `bioconductor/bioconductor_docker:RELEASE_X_Y`.
- If a pull request or push targets `devel`, `master`, `main`, or any feature branch, it will default to `bioconductor/bioconductor_docker:devel`.
- We will also expose an optional `bioc_version` input to allow users to explicitly override this automatic derivation if needed.

## Alternatives Considered

- **Hardcoding `devel`**: This is what previous iterations of the workflow did. It was rejected because it forces developers to test release bugfixes against unstable `devel` dependencies, complicating the backporting process.
- **Requiring manual input**: We could have forced maintainers to explicitly define the container tag in their `.github/workflows/check-bioc.yml` file. This was rejected because it adds boilerplate and increases the chance of human error (e.g., forgetting to update the tag when branching for a new release).
- **Matrix testing across multiple versions**: Testing against both `devel` and the current release simultaneously. This was rejected for the "Rapid Feedback Check" workflow to keep CI times fast. Maintainers wanting multi-environment testing are directed to the comprehensive R-Universe workflow instead.

## Consequences

- **Easier backports**: Developers can confidently push to `RELEASE_X_Y` branches knowing the CI will test against the correct environment.
- **Reduced false failures**: Package tests won't spuriously fail due to breaking changes in upstream `devel` dependencies when working on stable release branches.
- **Slightly more complex workflow**: The CI pipeline now requires a preliminary `set-matrix` job to compute the tag before the main `check` job can run. This adds a few seconds to the total CI runtime and makes the YAML slightly harder to read.
