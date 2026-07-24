# 0006. Self-Contained Canary Test Architecture

- **Status:** Accepted
- **Date:** 2026-07-24
- **Deciders:** Levi Waldron

## Context and Problem Statement

Reusable GitHub Actions workflows can silently break for downstream users when Bioconductor container images are updated, upstream R packages change their APIs, or pinned action versions are deprecated. Because `lwaldron/workflows` is used by many repositories, a regression here has a multiplied blast radius.

To detect such bitrot proactively, we need a scheduled integration test ("canary") that exercises `bioccheck.yml` end-to-end and alerts maintainers if it fails. However, `bioccheck.yml` is designed to check out and validate a caller repository as an R package (it runs `remotes::install_local(".")`, `rcmdcheck`, and `BiocCheck`). This means the canary must run within the context of a valid R package.

How do we structure the canary so it continuously validates the deployed workflow without introducing unnecessary complexity or external dependencies?

## Decision

We will make the `lwaldron/workflows` repository itself a minimal, valid R package by adding a `DESCRIPTION` file to its root. The canary workflow (`.github/workflows/canary.yml`) then calls `bioccheck.yml@main` — targeting the published `main` branch — with `error_on: "never"` to suppress expected BiocCheck notes on the stub package.

Key specifics of the decision:
- **`@main` reference**: The canary calls `uses: lwaldron/workflows/.github/workflows/bioccheck.yml@main` rather than the local `uses: ./.github/workflows/bioccheck.yml`. This ensures the scheduled run validates the *deployed* version of the workflow, not an unmerged branch.
- **`error_on: "never"`**: The stub `DESCRIPTION` intentionally has no vignettes, NEWS file, or ORCID. `BiocCheck` would emit notes for all of these. Setting `error_on: "never"` means the canary logs these expected notes without failing, isolating true environmental failures (broken containers, missing packages) from expected stub-related noise.

## Alternatives Considered

- **Separate canary repository**: A separate `lwaldron/workflows-canary` repository containing a real R package, with the cron scheduled there. This was rejected because it fragments maintenance across two repositories and adds overhead for every future change to the workflow (you would need to update or re-run the canary repo separately).
- **`working-directory` input on `bioccheck.yml`**: Add a new `working-directory` input to `bioccheck.yml` and place a dummy package in a `tests/` subdirectory of this repo. This was rejected because it would complicate the core reusable workflow for all downstream users solely to accommodate the canary use case.
- **Fail on `"warning"` in the canary**: Using the default `error_on: "warning"` in the canary would cause it to immediately fail due to BiocCheck warnings about the stub package (missing vignettes, etc.), making it impossible to distinguish real failures from expected ones.

## Consequences

- **Self-contained**: The entire canary lives in this repository. No external dependencies or separate repositories to maintain.
- **Accurate failure signal**: By using `error_on: "never"`, a canary failure unambiguously indicates a real environmental problem (e.g., a broken Bioconductor container), not a false positive from the stub package structure.
- **`lwaldron/workflows` is now an R package**: Future maintainers should be aware that the `DESCRIPTION` file is a deliberate sentinel artifact, not the beginning of a real package. This should be documented clearly in the `DESCRIPTION` itself and the README.
- **`@main` creates a one-commit lag**: Changes to `bioccheck.yml` are only tested by the canary after they land on `main`. PRs are validated by Actionlint (static) but not by a live canary run; this is an acceptable trade-off.
