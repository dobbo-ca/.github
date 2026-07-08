# dobbo-ca shared CI

Reusable release automation shared across dobbo-ca (and editorlint-org) repos.
Replaces per-repo `uplift` with `git-cliff`-based semantic versioning.

## What's here

- **`actions/semver-bump`** — composite action. Conventional-commit version resolution
  with git-cliff (pinned 2.13.1). Modes: `beta` (merge → `X.Y.Z-beta.<run#>` prerelease),
  `release` (a pushed semver tag *is* the version), `release-on-merge` (compute + tag a real
  `X.Y.Z` on merge), `auto` (tag ref ⇒ release, else beta). `dry-run: true` computes without tagging.
- **`.github/workflows/release-go.yml`** — reusable workflow (`workflow_call`) for pure-Go,
  cross-compiled binaries: semver-bump → build matrix → GitHub release. Homebrew/Chocolatey
  publishing stays in the caller as a small local job, gated on the `is_release` output.

## Consuming it

Pin `@v1`.

```yaml
# composite action
- uses: dobbo-ca/.github/actions/semver-bump@v1
  with: { mode: auto, dry-run: false }

# reusable workflow
jobs:
  release:
    if: ${{ !(github.ref_type == 'tag' && contains(github.ref_name, '-')) }}  # ignore bot beta tags
    uses: dobbo-ca/.github/.github/workflows/release-go.yml@v1
    with: { binary-name: mytool, cmd-path: ./cmd/mytool, build-matrix: '[...]', asset-name-template: 'mytool-{version}-{goos}-{goarch}' }
    secrets: { app-id: ${{ vars.GH_PUB_APP_CLIENT_ID }}, app-pem: ${{ secrets.GH_PUB_APP_PEM }} }
```

Test either path safely with `gh workflow run <wf> --ref <branch> -f dry_run=true -f mode=beta`.
