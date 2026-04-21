# PR size step (`actions/steps/pr-size`)

Fails PRs that exceed file/line-change thresholds unless an override marker is present.

## What it does

- Checks out the repo.
- Uses `gh pr view` to get additions/deletions/files/title/body.
- Computes total line changes and compares against thresholds.
- If the override regex matches the title/body, it emits a warning and passes.

## Inputs

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `overridePattern` | No | `\[pr-size-override\]` | Regex marker to allow an override |
| `maxChanges` | No | `100` | Maximum additions+deletions |
| `maxFiles` | No | `100` | Maximum files changed |

## Required permissions

- Requires `GH_TOKEN` (uses `${{ github.token }}`) to call the GitHub API via `gh`.
