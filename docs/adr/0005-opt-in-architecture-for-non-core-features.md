# 0005. Opt-in Architecture for Non-Core Features (pkgdown, Codecov)

- **Status:** Accepted
- **Date:** 2026-07-24
- **Deciders:** Bioconductor Core Team, Levi Waldron

## Context and Problem Statement

A modern continuous integration pipeline often does more than just run tests. It is common to deploy documentation websites (via `pkgdown`) and upload test coverage metrics (via `covr` and `codecov`). However, these auxiliary features inherently require elevated repository permissions (e.g., `pages: write` for GitHub Pages deployment) or external secrets (e.g., `CODECOV_TOKEN`). 

If a reusable workflow attempts to run these steps by default, it will inevitably fail for users who haven't performed the necessary out-of-band setup (generating tokens, altering repository settings). This creates friction and a poor out-of-the-box experience.

How should the workflow balance providing these powerful features while ensuring the core CI functionality works seamlessly for new users?

## Decision

We will design the workflow such that the core functionality (`R CMD check` and `BiocCheck`) "just works" by default with zero configuration. All non-core features requiring external configuration or elevated permissions are disabled by default and must be explicitly opted into.

Specifically:
- **`pkgdown` Deployment**: Controlled by the `enable_pkgdown` input, which defaults to `false`. Users must explicitly set it to `true` and ensure they grant `pages: write` and `id-token: write` permissions in their caller workflow.
- **Codecov Uploads**: The coverage step only runs if a `CODECOV_TOKEN` is explicitly passed via the `secrets` context. If no token is provided, the step is silently skipped.

## Alternatives Considered

- **Enabled by default**: We could have defaulted `enable_pkgdown` to `true`. This was rejected because it causes the workflow to immediately fail for any user who hasn't explicitly enabled GitHub Actions to deploy to Pages in their repository settings, violating the "just works" principle.
- **Separate workflows**: We could have split `pkgdown` deployment and coverage into entirely separate reusable workflows (e.g., `pkgdown.yml`, `coverage.yml`). This was rejected because it fragments the CI setup. It is more efficient to run these steps inside the same container where dependencies are already resolved and cached, rather than spinning up entirely new jobs that have to re-download dependencies.

## Consequences

- **Frictionless Onboarding**: A user can add the workflow to their repository with a simple two-line `uses:` statement and immediately get a working Bioconductor CI pipeline without reading extensive documentation on secrets or permissions.
- **Reduced Support Burden**: Maintainers will receive fewer "workflow failed" issues caused by misconfigured tokens or missing Pages permissions.
- **Slight Discoverability Cost**: Users might not realize these features are available without reading the `README.md`. To mitigate this, we clearly document the opt-in flags in the main documentation table.
