# Jenkins → GitHub Actions Migration Report

## Summary

The repository's Jenkins **scripted pipeline** (`Jenkinsfile`) has been migrated to a GitHub Actions
workflow at `.github/workflows/ci.yml`. The original pipeline built the project in parallel on three
Jenkins node labels (`precise`, `trusty`, `windows`) using a dynamically generated `builders` map and
the `parallel` step. This maps directly onto a GitHub Actions **matrix strategy**.

| Item | Value |
| --- | --- |
| Source file | `Jenkinsfile` (archived at `.github/ci-archive/Jenkinsfile`) |
| Pipeline type | Scripted (Groovy, `node {}` blocks inside `parallel`) |
| Target workflow | `.github/workflows/ci.yml` |
| Validation | `actionlint` v1.7.7 — passed with no findings |

## Conversion mapping

| Jenkins construct | GitHub Actions equivalent |
| --- | --- |
| `def labels = ['precise', 'trusty', 'windows']` + `parallel builders` | `strategy.matrix.include` with three entries (jobs run in parallel by default) |
| `node(label) { ... }` | `runs-on: ${{ matrix.runs-on }}` |
| `stage('Checkout') { checkout scm }` | `actions/checkout@v5.0.0` (SHA-pinned) |
| `stage('Build')` → `sh 'make'` / `bat 'build.bat'` | `run: ${{ matrix.build-command }}` with `defaults.run.shell` = `bash`/`cmd` |
| `stage('Test')` → `sh 'make test'` / `bat 'test.bat'` | `run: ${{ matrix.test-command }}` |
| `stage('Archive')` → `archiveArtifacts artifacts: 'build/**/*'` | `actions/upload-artifact@v7.0.1` with `path: build/**/*` |
| `isUnix()` branching | Per-matrix-entry `shell` and command values (no runtime branching needed) |
| Implicit "fail all on first failure" | `fail-fast: false` so every platform reports its own result |

### Runner label mapping

| Jenkins label | GitHub runner | Notes |
| --- | --- | --- |
| `precise` | `ubuntu-22.04` | `precise` was Ubuntu 12.04; no equivalent hosted runner exists, mapped to the oldest supported hosted Ubuntu image |
| `trusty` | `ubuntu-24.04` | `trusty` was Ubuntu 14.04; mapped to the current supported hosted Ubuntu image |
| `windows` | `windows-latest` | Uses `cmd` shell so the original `bat` steps run unchanged |

If the build genuinely requires those legacy OS versions, replace the runner with a
`container:` image (for example `ubuntu:12.04`) or a self-hosted runner carrying the matching label.

## Triggers

The scripted pipeline declared no `triggers` block (Jenkins jobs were triggered by SCM polling or
webhooks configured in the Jenkins UI). The workflow therefore uses the standard equivalents:

- `push` on all branches
- `pull_request`
- `workflow_dispatch` for manual runs

## Secrets, variables and credentials

No `withCredentials` blocks, credential IDs, environment variables, or build parameters are used by
the original pipeline. **No repository secrets or variables need to be configured** for this
migration.

## Shared libraries

The pipeline contains no `@Library` annotations or `vars/` shared-library calls, so no inline
expansion was required.

## Actions used

All actions are from the verified `actions` organisation and pinned to a commit SHA:

| Action | Version | SHA |
| --- | --- | --- |
| `actions/checkout` | v5.0.0 | `08c6903cd8c0fde910a37f88322edcfb5dd907a8` |
| `actions/upload-artifact` | v7.0.1 | `043fb46d1a93c77aae656e7c1c64a875d1fc6a0a` |

`permissions: contents: read` is set at the workflow level to follow least-privilege defaults.

## Manual follow-up / known differences

1. **Artifact fingerprinting** — Jenkins' `fingerprint: true` has no GitHub Actions equivalent.
   Artifacts are uploaded per platform (`build-precise`, `build-trusty`, `build-windows`) and are
   subject to the repository's artifact retention settings.
2. **Legacy OS parity** — see the runner label mapping above; verify the build works on the
   modern Ubuntu images or switch to containers/self-hosted runners.
3. **Missing build scripts** — the repository does not currently contain a `Makefile`,
   `build.bat`, or `test.bat`. These were expected to exist on the Jenkins agents; the workflow
   will fail until they are present, exactly as the Jenkins pipeline would have.
4. **Artifact upload strictness** — `if-no-files-found: error` mirrors Jenkins' default behaviour
   of failing the build when no artifacts match the pattern.

## Validation performed

```console
$ actionlint
# (no output — validation passed)
```

## Archived files

- `.github/ci-archive/Jenkinsfile` — the original scripted pipeline (removed from the repository root)
