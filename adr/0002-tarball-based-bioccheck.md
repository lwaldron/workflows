# ADR 0002: Use Tarball-Based Execution for BiocCheck

**Status:** Accepted

**Date:** 2026-07-24

## Context

`BiocCheck` can be run either on the raw source directory or on a built package tarball. The current (pre-change) implementation in `bioccheck.yml` passed `"."` (the raw source directory) directly to `BiocCheck::BiocCheck()`.

## Decision

We revert to tarball-based execution: `rcmdcheck::rcmdcheck()` is called with `check_dir = "check"` so that the package tarball is produced as a side-effect, and `BiocCheck::BiocCheck()` is then pointed at that tarball via `dir('check', 'tar.gz$', full.names = TRUE)`.

## Rationale

1. **Fidelity to Bioconductor**: The official Bioconductor Build System runs `BiocCheck` on the built tarball, not the raw source directory. Running on the tarball ensures we catch the same issues that Bioconductor's own CI would catch.

2. **`.Rbuildignore` compliance**: Scanning the raw source directory causes `BiocCheck` to parse files that are explicitly excluded by `.Rbuildignore` (e.g. `.github/`, scratch scripts, vignette source assets). This produces false-positive warnings and notes that package maintainers must then investigate and dismiss. Using the tarball respects the author's intent about what constitutes the distributed package.

## Consequences

- `rcmdcheck::rcmdcheck()` must be called with `check_dir = "check"` to ensure the tarball is written to a known location.
- `BiocCheck::BiocCheck()` receives the path to the tarball rather than `"."`.
- The `no-check-bioc-help` flag is added to suppress the check for a Bioconductor support-site post (not relevant for non-Bioconductor-submitted packages using this workflow).
- Behaviour is now consistent with the legacy `waldronlab/bioc-pr-cmdcheck-pkgdown.yml` workflow.
