# Copilot Instructions for bioconductor/workflows

You are an expert in GitHub Actions and R/Bioconductor continuous integration. When assisting with this repository, adhere to the following rules:

## GitHub Actions Best Practices
1. **Reserved Secrets**: Never define `GITHUB_TOKEN` under `workflow_call: secrets:`. It is a reserved identifier. Use the native `${{ github.token }}` context instead.
2. **Secrets in Conditionals**: Never use `secrets.*` directly in an `if:` conditional. Map the secret to a step-level environment variable first, then evaluate the environment variable.

## R Scripting
1. **R.version Types**: `R.version$major` and `R.version$minor` are character objects, not integers. When using `sprintf`, use `%s` (e.g., `sprintf("R-%s", R.version$major)`).
2. **Bioconductor Environment**: The primary purpose of these workflows is testing R packages for Bioconductor. When executing R code in containers, expect a `bioconductor/bioconductor_docker` base.

## Architectural Guidelines
1. **Opt-in Architecture**: Core functionality must work with zero configuration. Any features requiring elevated repository permissions (e.g., `pages: write`) or external secrets (e.g., `CODECOV_TOKEN`) must be disabled by default and explicitly opted into via boolean inputs (see ADR 0005).
2. **Workflow Generator Synchronization**: Any changes made to `bioccheck.yml` inputs, secrets, or permissions MUST be mirrored in **both** the JavaScript object logic inside `docs/index.html` and the "Workflow Inputs & Parameters" reference table in `bioccheck_usage.md`. If a new feature requires repository-level setup (e.g., adding a secret), those instructions must be added to `bioccheck_usage.md` and linked directly from the option's help text in `docs/index.html` to prevent synchronization drift (see ADR 0007).
3. **Architecture Decision Records (ADRs)**: Any significant structural change or new design pattern in these reusable workflows should be documented with a new ADR in `docs/adr/` using the Nygard format.
4. **Release & Version Tag Synchronization**: Whenever changing the default workflow version reference (e.g. from `@main` to `@v1` or incrementing version tags), you MUST update all 4 reference locations in lockstep: `ci.yml`, `docs/index.html` (`workflowObj.jobs.bioccheck.uses`), `README.md`, and `bioccheck_usage.md`. See `technical_notes.md` for the full release checklist.
