# 0002. Reject Source-Based Execution of BiocCheck

- **Status:** Rejected
- **Date:** 2026-07-24
- **Deciders:** Levi Waldron

## Context and Problem Statement

`BiocCheck` is the standard tool for verifying that an R package conforms to Bioconductor guidelines. Some community CI workflows (such as `waldronlab/bioc-pr-cmdcheck-pkgdown.yml`) first run `R CMD build` to produce a compressed source tarball (`.tar.gz`), and then execute `BiocCheck` against that built tarball. Other community members (such as https://github.com/grimbough/bioc-actions/blob/v1-branch/run-BiocCheck/action.yml) have simplified the workflow by running `BiocCheck` directly on the raw source code directory.

We considered simplifying the GitHub Actions workflow by stripping out the logic required to locate the built tarball and instead running `BiocCheck(".")` directly on the repository source.

Is it acceptable to run `BiocCheck` on the raw source directory instead of the `.tar.gz` tarball in order to simplify the CI pipeline?

## Decision

We will **reject** the simplification of running `BiocCheck(".")` on the raw source. We will continue to run `BiocCheck` on the built `.tar.gz` tarball artifact, mirroring `waldronlab/bioc-pr-cmdcheck-pkgdown.yml` and the Bioconductor
Build System.

In the GitHub Actions workflow, this is implemented by forcing `rcmdcheck` to output its build artifacts to a known directory (`check_dir = "check"`), and then explicitly targeting the tarball:
```R
          BiocCheck::BiocCheck(
            dir('check', 'tar.gz$', full.names = TRUE),
            `quit-with-status` = TRUE, `no-check-bioc-help` = TRUE
          )
```

## Alternatives Considered

- **Checking the Source Directory (`BiocCheck(".")`)**: This was the initial proposal to simplify the YAML. However, it was rejected for two critical reasons:
  1. **`.Rbuildignore` bypass**: When building a tarball, R actively excludes files matching the `.Rbuildignore` regex (e.g., `.github/`, local dev scripts, raw datasets). If `BiocCheck` runs on the raw source directory, it scans these non-distributed files, which frequently leads to false positive warnings or notes (e.g., complaining about file sizes or unconventional files that won't even be present in the final package).
  2. **BBS Fidelity**: The official Bioconductor Build System (BBS) evaluates packages by building a tarball and running `BiocCheck` on that tarball. Running on the raw source directory introduces an environmental mismatch between local CI and official CRAN/Bioconductor servers.

## Consequences

- **100% Fidelity**: The CI pipeline exactly mirrors what happens on the Bioconductor servers.
- **Accurate Error Reporting**: Package maintainers will not be bogged down with false positive warnings about files that are correctly ignored by `.Rbuildignore`.
- **Minor YAML Complexity**: We must retain the slightly verbose `dir('check', ...)` regex in the workflow to locate the dynamically named tarball.
