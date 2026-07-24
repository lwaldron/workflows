# 0003. Configurable `rcmdcheck` Severity (`error_on`)

- **Status:** Accepted
- **Date:** 2026-07-24
- **Deciders:** Bioconductor Core Team, Levi Waldron

## Context and Problem Statement

`rcmdcheck::rcmdcheck()` executes the standard `R CMD check` process. It takes an argument `error_on` which determines what level of issue causes the function to throw an error (and thus fail the CI pipeline). Options are `"never"`, `"note"`, `"warning"`, or `"error"`.

Upstream legacy workflows often hardcoded this to `"error"`. However, different package maintainers have different policies. Some prefer strict adherence (failing on any warning or even note), while others working on legacy or rapid-iteration codebases might prefer to only fail on hard errors. 

How should the reusable workflow handle the severity threshold for `R CMD check`?

## Decision

We will expose `error_on` as an optional input parameter in the reusable workflow (`bioccheck.yml`), with a default value of `"warning"`.

In the workflow schema:
```yaml
      error_on:
        description: 'Error policy for rcmdcheck (never, note, warning, error)'
        required: false
        type: string
        default: 'warning'
```

## Alternatives Considered

- **Hardcoding to `"error"`**: This is the default behavior in many vanilla R CI templates. However, it was rejected because it removes agency from the package maintainer. It also causes the CI to pass even if the check emits serious warnings, which is generally discouraged in Bioconductor.
- **Hardcoding to `"warning"`**: While this is the recommended standard for most packages (failing on warnings ensures high code quality), hardcoding it was rejected. Maintainers might have temporary, unavoidable warnings during heavy refactoring or when upstream dependencies introduce deprecations, and need a way to temporarily relax the CI checks.

## Consequences

- **Flexibility**: Package maintainers can dial the strictness up (`"note"`) or down (`"error"`) depending on the maturity and context of their package.
- **High Defaults**: By defaulting to `"warning"`, we encourage best practices out of the box without permanently locking maintainers out of successful CI runs if they choose to override it.
- **Complexity**: Requires a small amount of bash parsing/validation in the workflow to ensure the input is one of the four allowed strings before passing it to R.
