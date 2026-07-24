# Technical Notes & Software Architecture

This document describes the software architecture and internal maintenance details of the `bioconductor/workflows` repository.

## Architecture Decision Records (ADRs)

Any significant structural change or new design pattern in these reusable workflows is documented in the `docs/adr/` directory using the Nygard format.

Please refer to the [ADR Directory](docs/adr/README.md) for the complete list of Architecture Decision Records that guide the architecture.

## Copilot Instructions

When contributing to this repository using AI assistants, please review the rules and guidelines specified in [`.github/copilot-instructions.md`](.github/copilot-instructions.md).

## Internal Workflows

### [`actionlint.yml`](.github/workflows/actionlint.yml)
This repository uses `raven-actions/actionlint` to validate the GitHub Actions workflow syntax. It runs automatically on push and pull requests for any changes within `.github/workflows/`.

### [`canary.yml`](.github/workflows/canary.yml)
The canary workflow runs on a weekly schedule (and on-demand) to verify that `bioccheck.yml` executes correctly end-to-end against the Bioconductor container tag derived from the main branch. 
It uses `error_on: 'never'` so that expected BiocCheck notes from the internal stub package do not cause a false failure.
