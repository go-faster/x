# x

Go faster [reusable workflows](https://docs.github.com/en/actions/learn-github-actions/reusing-workflows).

## Caveats

1) Can't customize test matrix (e.g. add go version or platform)

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
jobs:
  test:
    uses: go-faster/x/.github/workflows/test.yml@main
  cover:
    uses: go-faster/x/.github/workflows/cover.yml@main
  lint:
    uses: go-faster/x/.github/workflows/lint.yml@main
  commit:
    uses: go-faster/x/.github/workflows/commit.yml@main
  nancy:
    uses: go-faster/x/.github/workflows/nancy.yml@main
  codeql:
    uses: go-faster/x/.github/workflows/codeql.yml@main
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