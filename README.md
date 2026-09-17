# github-platform-workflows

Central repo of reusable GitHub Actions workflows and composite actions.
Individual repos call these instead of duplicating CI/CD/GHAS logic.

## Setup

1. Create this repo on GitHub, e.g. `github.com/<your-username>/github-platform-workflows`.
2. Push all these files as-is.
3. In every workflow/action file below, replace `<your-github-username>` with your
   actual GitHub username (or org name).
4. If this repo is **private**, go to
   `Settings -> Actions -> General -> Access` and allow
   "Accessible from repositories in the same organization/account" so other
   repos you own can call these reusable workflows. If it's public, no
   change needed.

## What's inside

| File | Type | Purpose |
|---|---|---|
| `.github/actions/ci-build` | Composite action | Build / compile-check steps |
| `.github/actions/ci-unit-test` | Composite action | Install dependencies and run pytest |
| `.github/actions/codeql-init` | Composite action | Initializes CodeQL before build and test |
| `.github/actions/codeql-analyze` | Composite action | Analyzes and uploads CodeQL results |
| `.github/workflows/ci-build.yml` | Reusable workflow | Runs the build composite action |
| `.github/workflows/ci-unit-test.yml` | Reusable workflow | Runs the unit-test composite action |
| `.github/workflows/ghas-analyze.yml` | Reusable workflow | Runs the CodeQL composite action |
| `.github/workflows/dependency-review.yml` | Reusable workflow | Fails PRs that introduce vulnerable dependencies |
| `.github/workflows/secret-scan.yml` | Reusable workflow | Gitleaks-based secret scan (see note below) |
| `.github/actions/setup-python-env` | Composite action | Shared Python setup + pip cache, used *inside* the jobs above |

The reusable workflow files are retained as compatibility wrappers. New
consumer repositories should prefer one entry-point workflow that calls the
composite actions directly, so CodeQL initialization, build, tests, and
analysis run in one job and in a controlled order.

## Important gotcha: composite action references

Reusable workflows execute using **the calling repo's checked-out code**, not
this repo's files — even though the workflow YAML itself lives here. So a
relative path like `uses: ./.github/actions/setup-python-env` will NOT work
inside these reusable workflows. You must fully qualify it:

```yaml
uses: <your-github-username>/github-platform-workflows/.github/actions/setup-python-env@main
```

This is already done correctly in the reusable workflows and composite actions.

## Calling composite actions directly

The recommended integration is one entry-point workflow in each consumer
repository. It passes its parameters to these composite actions and controls
the order of operations:

```yaml
jobs:
  ci:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      actions: read
      security-events: write
    steps:
      - uses: actions/checkout@v4

      - uses: kprasadpn/github-platform-workflows/.github/actions/codeql-init@main
        with:
          language: python
          build-mode: none

      - uses: kprasadpn/github-platform-workflows/.github/actions/ci-build@main
        with:
          app-name: my-app
          python-version: "3.11"

      - uses: kprasadpn/github-platform-workflows/.github/actions/ci-unit-test@main
        with:
          python-version: "3.11"

      - uses: kprasadpn/github-platform-workflows/.github/actions/codeql-analyze@main
        with:
          language: python
```

The consumer workflow owns the runner, checkout, permissions, inputs, and
ordering. The shared repository owns the implementation of each build, test,
and CodeQL phase. For compiled languages, use the appropriate CodeQL build mode
and ensure the consumer's build action performs the required build between
`codeql-init` and `codeql-analyze`.

## Note on secret scanning

GitHub's **native** secret scanning doesn't need a workflow at all — it's a
repo setting (`Settings -> Security -> Code security and analysis ->
Secret scanning`). It's free and automatic on public repos; private repos
need GitHub Advanced Security (GHAS) enabled. The `secret-scan.yml` workflow
here (Gitleaks) is a workflow-based alternative/backup that works regardless
of GHAS licensing, useful if you want scan results as a PR check rather than
only in the Security tab.

## Versioning

For real use, tag releases (`git tag v1.0.0`) and have consumer repos pin to
`@v1.0.0` instead of `@main`, so template changes don't silently break every
repo at once. `@main` is fine for this test.
