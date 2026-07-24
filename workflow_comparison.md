# Comparison & Diff: `bioccheck.yml` vs `waldronlab/bioc-pr-cmdcheck-pkgdown.yml`

This document outlines the design decisions and technical differences between the new **`lwaldron/workflows/.github/workflows/bioccheck.yml`** workflow and the upstream **`waldronlab/.github/.github/workflows/bioc-pr-cmdcheck-pkgdown.yml`** workflow.

---

## 1. Summary of Key Differences

| Feature / Aspect | `waldronlab` Source Workflow | `lwaldron` (`bioccheck.yml`) | Key Advantage / Reason |
| :--- | :--- | :--- | :--- |
| **Container Tag (`DOCKER_TAG`)** | Hardcoded to `"devel"` unconditionally | **Dynamic tag resolution**: maps `RELEASE_X_Y` branch targets to `bioconductor_docker:RELEASE_X_Y` | Ensures PRs/pushes targeting Bioconductor release branches test against the matching release container rather than `devel`. |
| **Branch Override** | None | Optional `bioc_version` input parameter | Allows callers to manually override the container tag (e.g. `bioc_version: "RELEASE_3_20"`). |
| **`rcmdcheck` Severity (`error_on`)** | Hardcoded to `"error"` | **Configurable input `error_on`** (default: `"warning"`) | Allows workflows to strictly fail on warnings by default, or be set to `"error"` or `"never"`. |
| **Package Installation Method** | `remotes::install_deps()` + `BiocManager::install()` | `remotes::install_local(".", dependencies = TRUE)` | Explicitly builds and installs the local package under test in addition to installing its dependencies. |
| **`pkgdown` Deployment** | Always enabled by default (`enable_pkgdown: true`) | **Opt-in (`enable_pkgdown: false` default)** + GitHub Pages deployment | Core CI functionality works out-of-the-box without requiring GitHub Pages setup or elevated permissions unless enabled (ADR 0005). |
| **Docker Hub Image Push (`dock` job)** | Includes a job to build and push Docker images to Docker Hub | Omitted (removed unused `enable_docker` input) | Can add as an opt-in feature later. |

---

## 2. Full Unified Diff

Below is the complete unified diff comparing `waldronlab/bioc-pr-cmdcheck-pkgdown.yml` (left/minus) with `lwaldron/workflows/.github/workflows/bioccheck.yml` (right/plus):

```diff
--- waldronlab/bioc-pr-cmdcheck-pkgdown.yml
+++ lwaldron/workflows/.github/workflows/bioccheck.yml
@@ -1,4 +1,4 @@
-name: Bioc PR CMD check & (optional) pkgdown + Docker
+name: Bioc Container Check
 
 on:
   workflow_call:
@@ -19,26 +19,19 @@
         description: "If true, build + deploy pkgdown when on a RELEASE_* branch push"
         required: false
         type: boolean
-        default: true
-      enable_docker:
-        description: "If true, build/push docker when Dockerfile exists and on devel push"
-        required: false
-        type: boolean
-        default: true
-      dockerfile_path:
-        description: "Path to Dockerfile to build (if present)"
-        required: false
-        type: string
-        default: "inst/docker/pkg/Dockerfile"
+        default: false
+      error_on:
+        description: 'Error policy for rcmdcheck (never, note, warning, error)'
+        required: false
+        type: string
+        default: 'warning'
+      bioc_version:
+        description: 'Override Bioconductor version/container tag (e.g. devel, RELEASE_3_20). Auto-derived if empty.'
+        required: false
+        type: string
+        default: ''
     secrets:
       CODECOV_TOKEN:
+        description: 'Token for uploading coverage to Codecov'
         required: false
-      DOCKERHUB_USERNAME:
-        required: false
-      DOCKERHUB_TOKEN:
-        required: false
-
-env:
-  R_REMOTES_NO_ERRORS_FROM_WARNINGS: true
-  GITHUB_PAT: ${{ secrets.GITHUB_TOKEN }}
-  CRAN: ${{ inputs.cran }}
-
-  CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
-  DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
-  DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
 
 jobs:
   set-matrix:
-    runs-on: ubuntu-24.04
+    runs-on: ubuntu-latest
     outputs:
-      matrix: ${{ steps.set.outputs.matrix }}
-      dockerfile_exists: ${{ steps.dockerfile.outputs.exists }}
       is_release_branch: ${{ steps.branch.outputs.is_release_branch }}
-      ref_name: ${{ steps.branch.outputs.ref_name }}
+      docker_tag: ${{ steps.matrix.outputs.docker_tag }}
+      checkout_ref: ${{ steps.matrix.outputs.checkout_ref }}
     steps:
-      - name: Determine ref name + release-branch status
-        id: branch
+      - name: Set Matrix Variables
+        id: matrix
         shell: bash
         run: |
-          # For PRs, use the target branch (base_ref). For pushes, use ref_name.
+          OVERRIDE_TAG="${{ inputs.bioc_version }}"
+
+          if [ -n "$OVERRIDE_TAG" ]; then
+            DOCKER_TAG="$OVERRIDE_TAG"
+            REF_NAME="${GITHUB_REF_NAME}"
+          else
+            # For PRs, use the target base branch; for pushes, use the pushed branch.
             if [[ "${GITHUB_EVENT_NAME}" == "pull_request" ]]; then
               REF_NAME="${{ github.base_ref }}"
             else
               REF_NAME="${GITHUB_REF_NAME}"
             fi
 
-          echo "ref_name=${REF_NAME}" >> "$GITHUB_OUTPUT"
-
+            # Derive the container tag from the branch name.
+            if [[ "${REF_NAME}" =~ ^RELEASE_ ]]; then
+              DOCKER_TAG="${REF_NAME}"
+            elif [[ "${REF_NAME}" == "devel" || "${REF_NAME}" == "master" || "${REF_NAME}" == "main" ]]; then
+              DOCKER_TAG="devel"
+            else
+              DOCKER_TAG="devel"
+            fi
+          fi
+
           if [[ "${REF_NAME}" =~ ^RELEASE_ ]]; then
-            echo "is_release_branch=true" >> "$GITHUB_OUTPUT"
+            IS_RELEASE="true"
           else
-            echo "is_release_branch=false" >> "$GITHUB_OUTPUT"
+            IS_RELEASE="false"
           fi
 
-      - name: Set Matrix (docker tag + checkout ref)
-        id: set
-        shell: bash
-        run: |
-          DOCKER_TAG="devel"
-
           if [[ "${GITHUB_EVENT_NAME}" == "pull_request" ]]; then
             CHECKOUT_REF="${{ github.event.pull_request.head.sha }}"
           else
-            CHECKOUT_REF="${GITHUB_REF_NAME}"
+            CHECKOUT_REF="${GITHUB_SHA}"
           fi
 
-          MATRIX="{\"include\":[{\"docker_tag\":\"${DOCKER_TAG}\",\"checkout_ref\":\"${CHECKOUT_REF}\"}]}"
-          echo "matrix=$MATRIX" >> "$GITHUB_OUTPUT"
-
-      - name: Check for Dockerfile
-        id: dockerfile
-        shell: bash
-        run: |
-          if [ -f "./${{ inputs.dockerfile_path }}" ]; then
-            echo "exists=true" >> "$GITHUB_OUTPUT"
-          else
-            echo "exists=false" >> "$GITHUB_OUTPUT"
-          fi
-
   check:
     needs: set-matrix
     runs-on: ubuntu-latest
-    strategy:
-      fail-fast: false
-      matrix: ${{ fromJson(needs.set-matrix.outputs.matrix) }}
-
-    container: ghcr.io/bioconductor/bioconductor_docker:${{ matrix.docker_tag }}
-
+    container:
+      image: bioconductor/bioconductor_docker:${{ needs.set-matrix.outputs.docker_tag }}
+    env:
+      R_REMOTES_NO_ERRORS_FROM_WARNINGS: 'true'
+      GITHUB_PAT: ${{ github.token }}
+      _R_CHECK_CRAN_INCOMING_REMOTE_: 'false'
+      CODECOV_TOKEN: ${{ secrets.CODECOV_TOKEN }}
     steps:
       - name: Checkout Repository
         uses: actions/checkout@v4
         with:
-          ref: ${{ matrix.checkout_ref }}
+          ref: ${{ needs.set-matrix.outputs.checkout_ref }}

-      - name: Query dependencies
-        run: |
-          options(repos = c(CRAN = Sys.getenv("CRAN")))
-          BiocManager::install(c("remotes", "rcmdcheck", "BiocCheck", "covr"))
-          saveRDS(remotes::dev_package_deps(dependencies = TRUE), ".github/depends.Rds", version = 2)
+      - name: Query R and System Dependencies
+        id: get-deps
         shell: Rscript {0}
+        run: |
+          options(repos = c(CRAN = "https://packagemanager.posit.co/cran/__linux__/jammy/latest", BiocManager::repositories()))
+          if (!requireNamespace("remotes", quietly = TRUE)) install.packages("remotes")
+          deps <- remotes::dev_package_deps(".", dependencies = TRUE)
+          saveRDS(deps, ".github/depends.Rds")
+          writeLines(sprintf("R-%s.%i", R.version$major, floor(as.numeric(R.version$minor))), ".github/R-version")

-      - name: Cache R packages
+      - name: Cache R Packages
         uses: actions/cache@v4
         with:
           path: /usr/local/lib/R/site-library
-          key: ${{ runner.os }}-r-${{ matrix.checkout_ref }}-${{ hashFiles('.github/depends.Rds') }}
-          restore-keys: |
-            ${{ runner.os }}-r-${{ matrix.checkout_ref }}-
+          key: ${{ runner.os }}-${{ hashFiles('.github/R-version') }}-1-${{ hashFiles('.github/depends.Rds') }}
+          restore-keys: ${{ runner.os }}-${{ hashFiles('.github/R-version') }}-1-

-      - name: Install GPG
-        if: ${{ github.ref == 'refs/heads/devel' && github.event_name != 'pull_request' }}
-        run: sudo apt-get update && sudo apt-get install -y gpg
-
-      - name: Install Dependencies
-        run: |
-          options(repos = c(CRAN = Sys.getenv("CRAN")))
-          remotes::install_deps(dependencies = TRUE, repos = BiocManager::repositories())
-          BiocManager::install(ask = FALSE, update = TRUE)
+      - name: Install Package Dependencies
         shell: Rscript {0}
+        run: |
+          options(repos = c(CRAN = "https://packagemanager.posit.co/cran/__linux__/jammy/latest", BiocManager::repositories()))
+          remotes::install_local(".", dependencies = TRUE, Ncpus = parallel::detectCores())
+          BiocManager::install(c("rcmdcheck", "BiocCheck", "covr", "pkgdown"), Ncpus = parallel::detectCores())

-      - name: Check Package
-        env:
-          _R_CHECK_CRAN_INCOMING_REMOTE_: false
-        run: rcmdcheck::rcmdcheck(args = "--no-manual", error_on = "error", check_dir = "check")
+      - name: Run R CMD check
+        env:
+          RCMDCHECK_ERROR_ON: ${{ inputs.error_on }}
         shell: Rscript {0}
-
-      - name: Test coverage
-        if: ${{ success() && github.ref == 'refs/heads/devel' && github.event_name != 'pull_request' }}
         run: |
-          cov <- covr::package_coverage(
-            quiet = FALSE,
-            clean = FALSE,
-            type = "all",
-            install_path = file.path(
-              normalizePath(Sys.getenv("RUNNER_TEMP"), winslash = "/"),
-              "package"
-            )
-          )
-          covr::to_cobertura(cov)
-        shell: Rscript {0}
+          allowed <- c("never", "note", "warning", "error")
+          error_on <- Sys.getenv("RCMDCHECK_ERROR_ON", unset = "warning")
+          if (!error_on %in% allowed) stop("Invalid error_on value: '", error_on, "'. Must be one of: ", paste(allowed, collapse = ", "))
+          res <- rcmdcheck::rcmdcheck(".", args = c("--no-manual", "--as-cran"), error_on = error_on, check_dir = "check")
+          print(res)

-      - name: Upload test results to Codecov
-        if: ${{ success() && github.ref == 'refs/heads/devel' && github.event_name != 'pull_request' && env.CODECOV_TOKEN != '' }}
-        uses: codecov/codecov-action@v4
-        with:
-          fail_ci_if_error: ${{ github.event_name != 'pull_request' && true || false }}
-          file: ./cobertura.xml
-          plugin: noop
-          disable_search: true
-          token: ${{ secrets.CODECOV_TOKEN }}
-
       - name: Run BiocCheck
         shell: Rscript {0}
         run: |
-          BiocCheck::BiocCheck(
-            dir('check', 'tar.gz$', full.names = TRUE),
-            `quit-with-status` = TRUE, `no-check-bioc-help` = TRUE
-          )
+          tarballs <- dir("check", pattern = "\\.tar\\.gz$", full.names = TRUE)
+          if (length(tarballs) == 0) stop("No tarball found in check/ directory")
+          BiocCheck::BiocCheck(tarballs[1], `quit-with-status` = TRUE)

       - name: Test Coverage
         if: success() && needs.set-matrix.outputs.is_release_branch == 'false' && env.CODECOV_TOKEN != ''
         shell: Rscript {0}
         run: |
           cov <- covr::package_coverage()
           covr::codecov(coverage = cov, token = Sys.getenv("CODECOV_TOKEN"))

       - name: Build pkgdown Site
         if: >-
           ${{
             success()
             && inputs.enable_pkgdown
             && github.event_name != 'pull_request'
             && needs.set-matrix.outputs.is_release_branch == 'true'
           }}
         shell: Rscript {0}
         run: |
           pkgdown::build_site()

+      - name: Upload pkgdown artifact
+        if: >-
+          ${{
+            success()
+            && inputs.enable_pkgdown
+            && github.event_name != 'pull_request'
+            && needs.set-matrix.outputs.is_release_branch == 'true'
+          }}
+        uses: actions/upload-pages-artifact@v3
+        with:
+          path: docs

   deploy:
     needs:
       - check
       - set-matrix
     if: >-
       ${{
         inputs.enable_pkgdown
         && github.event_name != 'pull_request'
         && needs.set-matrix.outputs.is_release_branch == 'true'
       }}
     permissions:
       pages: write
       id-token: write
     environment:
       name: github-pages
       url: ${{ steps.deployment.outputs.page_url }}
     runs-on: ubuntu-latest
     steps:
       - name: Deploy to GitHub Pages
         id: deployment
         uses: actions/deploy-pages@v4
```
