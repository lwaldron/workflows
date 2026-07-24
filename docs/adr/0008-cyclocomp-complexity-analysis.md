# 0008. Opt-in Cyclomatic Complexity Analysis (cyclocomp)

- **Status:** Accepted
- **Date:** 2026-07-24
- **Deciders:** Bioconductor Core Team, Levi Waldron

## Context and Problem Statement

Code complexity can be a significant barrier to maintenance and contribution in R packages. Cyclomatic complexity is a useful metric to identify functions that may be too complex. We want to provide this information to package developers to encourage refactoring of overly complex functions. However, computing and interpreting this metric shouldn't impede the primary goal of the CI workflow, which is to run checks efficiently and reliably.

How should we integrate cyclomatic complexity analysis into the `bioccheck` workflow without adding unnecessary overhead or causing false-positive CI failures?

## Decision

We will add an opt-in feature for cyclomatic complexity analysis using the `cyclocomp` package.

Specifically:
- **Opt-in (`enable_cyclocomp`)**: The feature is disabled by default to avoid cluttering artifacts for users who don't need it. Users must explicitly set `enable_cyclocomp: true` to run the analysis.
- **Unconditional Installation**: The `cyclocomp` package is always installed alongside other core dependencies like `rcmdcheck` and `covr`. Because `cyclocomp` is a small package with few dependencies, the installation cost is negligible. This avoids complex conditional R installation logic within the workflow step.
- **Advisory Only (`continue-on-error`)**: The cyclocomp analysis step runs with `continue-on-error: true`. It is treated purely as an advisory code quality metric, not a pass/fail gate. Failures in this step will never cause the overall workflow to fail.
- **Resilient Upload**: The resulting `cyclocomp_results.csv` artifact is uploaded using `if: always()`, ensuring that any partial results are available even if other parts of the job fail.

## Alternatives Considered

- **Run unconditionally**: We could have run the analysis by default on every PR. This was rejected to avoid generating unnecessary artifacts and noise for users who do not intend to use this metric.
- **Conditional Installation**: We could have installed `cyclocomp` only when `enable_cyclocomp` is true. This was rejected because injecting conditional logic into a single R script block that installs multiple packages adds fragility, whereas the cost of unconditionally installing it is very low.
- **Fail on high complexity**: We could have added a threshold to fail the workflow if a function exceeds a certain complexity score. This was rejected because it violates the principle of providing non-intrusive feedback and could frustrate developers with subjective metric failures.

## Consequences

- **Code Quality Insights**: Developers get easy access to complexity metrics without needing to run tools locally.
- **Stable CI**: The core CI process remains fast and stable, with zero risk of the new feature causing workflow failures.
- **Simple Maintenance**: Avoiding conditional package installation keeps the `bioccheck.yml` file simpler and easier to maintain.
