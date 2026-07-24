# Bioconductor GitHub Actions Workflows

[![Actionlint](https://github.com/bioconductor/workflows/actions/workflows/actionlint.yml/badge.svg)](https://github.com/bioconductor/workflows/actions/workflows/actionlint.yml)
[![Canary Check](https://github.com/bioconductor/workflows/actions/workflows/canary.yml/badge.svg)](https://github.com/bioconductor/workflows/actions/workflows/canary.yml)

This repository (`bioconductor/workflows`) hosts reusable GitHub Actions workflows for R/Bioconductor package development.

## Overview: Two Complementary Testing Patterns

Two complementary workflows are available depending on your testing goals:

| Purpose | Workflow Reference | Target Environment | Managed By |
| :--- | :--- | :--- | :--- |
| **1. Rapid Feedback Check** | `bioconductor/workflows/.github/workflows/bioccheck.yml@v1` | Linux (`bioconductor_docker` container) | Bioconductor Core Team / Maintainers |
| **2. Comprehensive Pre-Release Check** | `r-universe-org/workflows/.github/workflows/build.yml@v3` | Multi-OS (Linux, macOS, Windows, WebAssembly) + `BiocCheck` | R-Universe Team |

## Documentation

The documentation for these workflows has been separated into distinct files for clarity:

- [**BiocCheck Usage**](bioccheck_usage.md): Comprehensive instructions for using the `bioccheck.yml` reusable workflow, including test coverage and `pkgdown` setup.
- [**R-Universe Usage**](r-universe-org_usage.md): Instructions for using the comprehensive multi-platform R-Universe workflow.
- [**Workflow Comparison**](workflow_comparison.md): A detailed comparison of the two workflows to help you choose the right one for your needs.
- [**Technical Notes**](technical_notes.md): Software architecture, ADRs, internal test flows (`canary.yml`, `actionlint.yml`), and guidelines for contributors.
