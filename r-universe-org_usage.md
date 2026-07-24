# Comprehensive Multi-Platform Check (`r-universe-org`)

For multi-platform verification across Linux, macOS, and Windows prior to release or submission, use the official **R-Universe workflow** (`r-universe-org/workflows/.github/workflows/build.yml@v3`).

When `organization: bioconductor` is provided, R-Universe automatically invokes `BiocCheck` via `r-universe-org/actions/bioc-check@v12`.

## Example Usage (`.github/workflows/runiverse.yml` in your package repo)

```yaml
name: R-Universe Build & BiocCheck

on:
  workflow_dispatch:           # Manual trigger before release/submission
  push:
    branches: [ devel ]        # Run on pushes to devel

jobs:
  build:
    uses: r-universe-org/workflows/.github/workflows/build.yml@v3
    with:
      universe: ${{ github.repository_owner }}
      organization: bioconductor
```

For additional configuration options and inputs, refer to the upstream [r-universe-org/workflows repository](https://github.com/r-universe-org/workflows).
