# x

Go faster [reusable workflows]https://docs.github.com/en/actions/learn-github-actions/reusing-workflows.

## Caveats

1) Can't customize test matrix (e.g. add go version or platform)
2) Can't skip workflows

## Example

Add `.github/workflows/ci.yml`
```yaml
name: x

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

# See https://github.com/go-faster/x
# Includes test, coverage, golangci-lint, commit lint, generate checks
jobs:
  ci:
    uses: go-faster/x/.github/workflows/ci.yml@main
```

Add `.github/workflows/pkg.yml`
```yaml
name: pkg

on:
  push:
    tags:
      - v*

# See https://github.com/go-faster/x
# Automates pkg.go.dev module release fetch.
jobs:
  run:
    uses: go-faster/x/.github/workflows/release.yml@main
```