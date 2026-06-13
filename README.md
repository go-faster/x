# x

Go faster [reusable workflows](https://docs.github.com/en/actions/learn-github-actions/reusing-workflows).

## Quick start

Add `.github/workflows/ci.yml`:

```yaml
name: ci

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

# Cancel in-progress runs on new push to the same branch/PR.
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    uses: go-faster/x/.github/workflows/test.yml@main
    permissions:
      contents: read
  cover:
    uses: go-faster/x/.github/workflows/cover.yml@main
    permissions:
      contents: read
  lint:
    uses: go-faster/x/.github/workflows/lint.yml@main
    permissions:
      contents: read
  commit:
    uses: go-faster/x/.github/workflows/commit.yml@main
    permissions:
      contents: read
      pull-requests: read
  govulncheck:
    uses: go-faster/x/.github/workflows/govulncheck.yml@main
    permissions:
      contents: read
      security-events: write
  codeql:
    uses: go-faster/x/.github/workflows/codeql.yml@main
    permissions:
      actions: read
      contents: read
      security-events: write
```

## Repository requirements

| Workflow | Requirement |
|---|---|
| All | `go.mod` and `go.sum` at the repo root |
| `cover.yml` (default) | `Makefile` with a `coverage` target that writes `profile.out` |
| `lint.yml` `enable-gen` | `go generate ./...` produces a clean diff (no uncommitted changes) |
| `lint.yml` `enable-mod` | `go mod tidy` produces a clean diff |
| `codeql.yml` | Repository must have the Security tab enabled |
| `govulncheck.yml` | `security-events: write` permission to report to the Security tab (optional — findings still print to the log without it) |

### Minimal `Makefile` for `cover.yml`

```makefile
.PHONY: coverage
coverage:
	go test -coverprofile=profile.out ./...
```

Or pass a custom command to skip the Makefile requirement entirely:

```yaml
cover:
  uses: go-faster/x/.github/workflows/cover.yml@main
  permissions:
    contents: read
  with:
    coverage-cmd: go test -coverprofile=profile.out ./...
```

## Security

### Pin to a commit SHA, not a tag

These workflows pin every external action to an immutable commit SHA.
Tags are mutable and can be silently redirected to malicious code after a supply-chain compromise.
The tag comment (`# v6`) identifies which release the hash corresponds to.

### Updating pinned SHAs

[Dependabot](https://docs.github.com/en/code-security/dependabot) is configured in this repo
(`dependabot.yml`) and opens PRs automatically when new action versions are released,
keeping both the SHA and the tag comment in sync. **No manual work needed in normal operation.**

To update a specific action manually:

```sh
# 1. Resolve the tag to its object SHA
gh api repos/OWNER/REPO/git/ref/tags/vX --jq '.object | {sha, type}'

# 2. If type == "tag" (annotated tag), dereference one more level
gh api repos/OWNER/REPO/git/tags/SHA --jq '.object.sha'

# 3. Replace the SHA in the workflow file; update the comment to match the new tag
```

Example — updating `actions/checkout`:

```sh
gh api repos/actions/checkout/git/ref/tags/v6 --jq '.object.sha'
# → df4cb1c...  (direct commit, type == "commit", done)
```

### Minimal permissions

Each job declares only the permissions it needs. Callers must grant at least those
permissions (see the quick-start example). The reusable workflow job's permissions are
**intersected** with the caller's grant — the job can never receive more than the caller
explicitly allows.

### Concurrency cancellation

The `concurrency` block in the quick-start example cancels redundant runs when new
commits are pushed to the same branch or PR. This prevents stale jobs from consuming
runner minutes and avoids cache-write races between parallel workflow runs.

## Available workflows

### `test.yml`

Runs `go test` across a matrix of OS/arch combinations.

| Input | Default | Description |
|---|---|---|
| `go` | `oldstable` | Go version |
| `runners` | `["ubuntu-latest","macos-latest","windows-latest"]` | JSON array of runners for the OS matrix |
| `enable-386` | `true` | Add linux/386 to the matrix |
| `enable-arm64` | `false` | Add linux/arm64 to the matrix |
| `submodules` | `true` | Checkout submodules |
| `test-args` | `""` | Extra `go test` flags (passed via env var, not interpolated into shell) |
| `timeout-minutes` | `30` | Job timeout |

Example — Linux only, race detector only, no submodules:

```yaml
test:
  uses: go-faster/x/.github/workflows/test.yml@main
  permissions:
    contents: read
  with:
    runners: '["ubuntu-latest"]'
    enable-386: false
    submodules: false
    test-args: "-count=1"
```

### `lint.yml`

Runs `golangci-lint` (including any formatter linters configured in `.golangci.yml`),
`go mod tidy` diff check, and `go generate` diff check as separate jobs.

| Input | Default | Description |
|---|---|---|
| `go` | `oldstable` | Go version |
| `golangci-lint` | `latest` | golangci-lint version |
| `golangci-lint-args` | `""` | Extra arguments for `golangci-lint run` |
| `enable-mod` | `true` | Run `go mod tidy` diff check |
| `enable-gen` | `true` | Run `go generate ./...` diff check |
| `runner` | `ubuntu-latest` | Runner for the main lint job |
| `runner-small` | `ubuntu-latest` | Runner for mod/gen jobs |
| `timeout-minutes` | `30` | Job timeout (also sets the `golangci-lint --timeout` flag) |

Example — skip generate check, pin lint version, use self-hosted runner for small jobs:

```yaml
lint:
  uses: go-faster/x/.github/workflows/lint.yml@main
  permissions:
    contents: read
  with:
    golangci-lint: v2.1.6
    enable-gen: false
    runner-small: self-hosted
```

### `cover.yml`

Runs a coverage build and uploads `profile.out` to Codecov.
The `coverage-cmd` is validated before execution: for `make <target>` commands the
target is verified to exist via `make -n`; for other commands the executable is checked
with `command -v`.

| Input | Default | Description |
|---|---|---|
| `go` | `oldstable` | Go version |
| `submodules` | `true` | Checkout submodules |
| `coverage-cmd` | `make coverage` | Command that produces `profile.out` |
| `timeout-minutes` | `30` | Job timeout for the test run |

### `govulncheck.yml`

Scans dependencies with [`govulncheck`](https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck).
Results appear in the Security tab when `security-events: write` is granted.

| Input | Default | Description |
|---|---|---|
| `go` | `oldstable` | Go version |
| `timeout-minutes` | `15` | Job timeout |

### `codeql.yml`

Runs GitHub CodeQL static analysis for Go.

| Input | Default | Description |
|---|---|---|
| `go` | `oldstable` | Go version |
| `timeout-minutes` | `30` | Job timeout |

### `commit.yml`

Lints commit messages against [Conventional Commits](https://www.conventionalcommits.org/)
using `commitlint`. Skipped automatically for Dependabot PRs.

### `release.yml`

Deprecated — triggers a pkg.go.dev module fetch on tag push.
