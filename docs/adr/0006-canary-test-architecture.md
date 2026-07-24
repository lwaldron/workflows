# 0006. Self-Contained Canary Test Architecture

- **Status:** Accepted
- **Date:** 2026-07-24
- **Deciders:** Levi Waldron

## Context and Problem Statement

Reusable GitHub Actions workflows can silently break for downstream users when Bioconductor container images are updated, upstream R packages change their APIs, or pinned action versions are deprecated. Because `lwaldron/workflows` is used by many repositories, a regression here has a multiplied blast radius.

To detect such bitrot proactively, we need a scheduled integration test ("canary") that exercises `bioccheck.yml` end-to-end and alerts maintainers if it fails. However, `bioccheck.yml` is designed to check out and validate a caller repository as an R package (it runs `remotes::install_local(".")`, `rcmdcheck`, and `BiocCheck`). This means the canary must run within the context of a valid R package.

How do we structure the canary so it continuously validates the deployed workflow without introducing unnecessary complexity or external dependencies?

## Decision

We will make the `lwaldron/workflows` repository itself a minimal, valid R package by adding a `DESCRIPTION` file to its root. The canary workflow (`.github/workflows/canary.yml`) then calls the local reusable workflow using relative path syntax `uses: ./.github/workflows/bioccheck.yml` with `error_on: "never"` so the `rcmdcheck` step does not fail on expected findings from the stub package.

Key specifics of the decision:
- **Relative Path Reference**: GitHub Actions strictly requires same-repository reusable workflows to be called using relative path syntax (`uses: ./.github/workflows/bioccheck.yml`). Using fully-qualified names (e.g. `owner/repo/path@ref`) for same-repository calls causes runtime "invalid workflow file" errors in GitHub Actions.
- **`error_on: "never"`**: This input only affects the `rcmdcheck` step. It keeps expected `rcmdcheck` findings from the stub package from failing the canary, while `BiocCheck` still runs normally and surfaces its expected notes in the logs.

## Alternatives Considered

- **Separate canary repository**: A separate `lwaldron/workflows-canary` repository containing a real R package, with the cron scheduled there. This was rejected because it fragments maintenance across two repositories and adds overhead for every future change to the workflow (you would need to update or re-run the canary repo separately).
- **`working-directory` input on `bioccheck.yml`**: Add a new `working-directory` input to `bioccheck.yml` and place a dummy package in a `tests/` subdirectory of this repo. This was rejected because it would complicate the core reusable workflow for all downstream users solely to accommodate the canary use case.
- **Fail on `"warning"` in the canary**: Using the default `error_on: "warning"` would make the synthetic stub package more likely to fail the `rcmdcheck` step on non-actionable findings, adding noise to a workflow whose main purpose is to detect environmental regressions.

## Consequences

- **Self-contained**: The entire canary lives in this repository. No external dependencies or separate repositories to maintain.
- **Lower-noise failure signal**: By using `error_on: "never"`, the canary is less likely to fail on expected `rcmdcheck` findings from the stub package, making environmental regressions easier to spot.
- **`lwaldron/workflows` is now an R package**: Future maintainers should be aware that the `DESCRIPTION` file is a deliberate sentinel artifact, not the beginning of a real package. This should be documented clearly in the `DESCRIPTION` itself and the README.
- **`@main` creates a one-commit lag**: Changes to `bioccheck.yml` are only tested by the canary after they land on `main`. PRs are validated by Actionlint (static) but not by a live canary run; this is an acceptable trade-off.
