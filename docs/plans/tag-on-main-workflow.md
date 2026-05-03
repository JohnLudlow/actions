---
title: Tag on Main Workflow
status: in-progress
issue: https://github.com/JohnLudlow/actions/issues/4
created: 2026-05-03
---

## Problem Statement

When this repository is referenced as a Git submodule in other repositories,
`git submodule status` shows only the raw commit SHA. This is opaque — there is
no human-readable label to indicate what version of the actions repo is in use.

Tagging each commit to `main` with its computed semantic version (e.g. `v1.4.2`)
would allow `git submodule status` to display the tag name alongside the SHA,
making submodule references much easier to read and reason about.

See [GitHub issue #4](https://github.com/JohnLudlow/actions/issues/4).

## History and Root Cause of Previous Failures

Two previous merge attempts failed with GitVersion errors. The confirmed root
cause in both cases was `workflow: Mainline` in `GitVersion.yml`.

GitVersion v6 introduced a `workflow:` key that references a named built-in
preset configuration. The valid preset names embedded in GitVersion v6 are:

- `GitFlow/v1`
- `GitHubFlow/v1`
- `TrunkBased/v1`

`Mainline` is **not** a valid v6 preset name. It was a deployment mode in
GitVersion v5. Using it in v6 produces:

```text
System.InvalidOperationException: Could not find embedded resource
GitVersion.Configuration.Workflows.Mainline.yml
```

The fix is to either use a valid preset name (e.g. `workflow: GitHubFlow/v1`)
or provide a fully explicit configuration without the `workflow:` key.

## Proposed Solution

Three files require changes:

| File | Change |
| --- | --- |
| `GitVersion.yml` | Replace with `workflow: GitHubFlow/v1` — the correct v6 preset |
| `steps/setup/action.yml` | Add `dotnet` input (default `'true'`) to make .NET steps conditional |
| `.github/workflows/main.yml` | New workflow file using `./steps/setup` with `dotnet: 'false'` |

## GitVersion Configuration File

Replace `GitVersion.yml` at the repository root with the single-line preset
reference:

```yaml
workflow: GitHubFlow/v1
```

`GitHubFlow/v1` is the embedded preset closest to the desired behaviour for
this repository. On the `main` branch it produces clean `X.Y.Z` version
numbers (no pre-release suffix) with a patch increment per commit. On feature
and PR branches it produces pre-release labels, which is useful for consuming
repos to distinguish tagged releases from development versions.

This is equivalent to — and replaced by — the 116-line explicit expansion
used in the previous iteration.

## Modifications to `steps/setup/action.yml`

Add a `dotnet` input with default `'true'` and condition the three .NET-specific
steps on it. This makes the action usable in non-.NET repositories (such as
this one) without failing on `dotnet restore`.

```yaml
inputs:
  dotnet-version:
    description: '.NET version to setup'
    default: '10.0.x'
  dotnet:
    description: 'Run .NET setup, NuGet cache, and restore steps'
    default: 'true'
```

The three steps to condition are: `Setup .NET`, `Cache NuGet packages`,
`Restore dependencies`. The default `'true'` preserves existing behaviour
for all callers that omit the input.

Use bare expression form for the `if:` condition, consistent with the rest of
the repository:

```yaml
if: inputs.dotnet == 'true'
```

## Workflow File

```yaml
name: Main

on:
  push:
    branches:
      - main
  pull_request:
  workflow_dispatch:

jobs:
  version:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    outputs:
      full-sem-ver: ${{ steps.setup.outputs.GitVersion_FullSemVer }}
    steps:
      - uses: actions/checkout@v5
        with:
          persist-credentials: false
          lfs: true
          submodules: recursive
          fetch-depth: 0

      - id: setup
        uses: ./steps/setup
        with:
          dotnet: 'false'

  tag:
    needs: version
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v5
        with:
          persist-credentials: true
          lfs: true
          submodules: recursive
          fetch-depth: 0

      - uses: ./steps/git/tag-commit
        with:
          version: ${{ needs.version.outputs.full-sem-ver }}
```

The `pull_request` and `workflow_dispatch` triggers allow the workflow to be
tested on a branch before merging. The `version` job runs on all triggers with
`contents: read`. The `tag` job only runs on `push` to `main`, where it
re-checks out the repository with `persist-credentials: true` and
`contents: write` in order to push the tag. The `tag-commit` action also has
an internal guard (`if: github.event_name == 'push' && github.ref ==
'refs/heads/main'`) so it will not create a tag on PR or manual-dispatch runs.

## Design Notes

### Why `./steps/setup` is used here (with `dotnet: 'false'`)

`./steps/setup` was designed for workflows that build .NET projects. It
unconditionally ran `dotnet restore`, which fails when no `.csproj` or `.sln`
file exists. The `dotnet` input added to `./steps/setup` makes the three
.NET-specific steps conditional, allowing this workflow to reuse the setup
action without .NET.

Using `./steps/setup` (rather than calling GitVersion actions directly) keeps
the pattern consistent with consuming repos such as GameEngineAdapter, and
ensures the same GitVersion version specification is used everywhere.

### Double-checkout tradeoff (version job only)

The outer checkout step (in the `version` job) and the inner checkout step
(inside `./steps/setup`) both run `actions/checkout@v5`. The inner checkout
re-fetches full history and resets submodule state set by the outer. This is a
known architectural tradeoff — not harmful for this workflow (which does not
need submodule content after setup) but worth noting:

- The outer checkout uses `persist-credentials: false`; the inner checkout
  inside `./steps/setup` uses the default (`persist-credentials: true`). No
  credential leak risk as neither job pushes from the `version` job.
- Wasted CI time: two fetches with `fetch-depth: 0` per run in the `version` job.

If minimising checkout time becomes a priority, a `skip-checkout` input could
be added to `./steps/setup` in a future change.

### Why `permissions` is scoped to each job

`permissions: contents: write` is placed only on the `tag` job. The `version`
job uses `contents: read`. This follows the principle of least privilege:
third-party actions (GitVersion) run under the read-only token, and the write
token is granted only to the job that actually pushes the tag.

### Why the `tag` job uses a separate checkout

The `tag` job does not call `./steps/setup` — it only needs to push a tag.
A second checkout with `persist-credentials: true` is required so that
`git push` can authenticate. Because `./steps/setup` is not called here,
there is no double-checkout overhead in the `tag` job.

### Why the workflow needs `contents: write`

The `tag-commit` step runs `git push origin <tag>`. Pushing to the repository
requires the `GITHUB_TOKEN` to have write access to repository contents.
GitHub Actions workflows default to read-only tokens, so the job must
explicitly grant `contents: write`.

### Why the workflow needs an explicit checkout before `./steps/git/tag-commit`

The workflow uses a local composite action (`./steps/git/tag-commit`). GitHub
Actions resolves local action paths relative to the workspace, but the
workspace is empty until the repository is checked out. The
`actions/checkout@v5` step at the top of the job ensures the local action
files are present before they are referenced.

### Workflow re-runs and the "tag already exists" failure mode

The `tag-commit` action runs `git tag $tag && git push origin $tag`. If a
workflow run is manually re-triggered after a successful run on the same
commit, `git tag` will exit with:

```text
fatal: tag 'vX.Y.Z' already exists
```

This causes the step — and the workflow — to fail visibly. This is the
intended behaviour: fail loudly rather than silently succeed or overwrite an
existing tag. Operators who re-trigger a run on the same commit should expect
this failure and can dismiss it. If force-updating tags or graceful skipping
is required in future, that change belongs in `steps/git/tag-commit`, not in
this workflow.

### Checkout options

`version` job outer checkout:

| Option | Value | Reason |
| --- | --- | --- |
| `persist-credentials` | `false` | No push needed from this job; avoids granting credentials to third-party actions. |
| `lfs` | `true` | Not currently used in this repo; included as a forward-compatible precaution. |
| `submodules` | `recursive` | This repo has no submodules; option is inert but included for consistency. |
| `fetch-depth` | `0` | Fetches full history. Required by GitVersion to compute the correct version. |

`tag` job checkout:

| Option | Value | Reason |
| --- | --- | --- |
| `persist-credentials` | `true` | Ensures `git push` can authenticate when pushing the tag. |
| `lfs` | `true` | Not currently used in this repo; included as a forward-compatible precaution per `tag-commit` docs. |
| `submodules` | `recursive` | This repo has no submodules; option is inert but included per `tag-commit` docs. |
| `fetch-depth` | `0` | Fetches full history. Required by `tag-commit` docs. |

See [`docs/git-tag-commit-step.md`](../git-tag-commit-step.md) for the full reference.

### The `tag-commit` action has an internal guard

The `tag-commit` composite action already contains an internal `if` condition
that restricts execution to `push` events on `main`:

```yaml
if: github.event_name == 'push' && github.ref == 'refs/heads/main'
```

The workflow trigger also restricts to `push` on `main`, making the guard
redundant in this context. Both constraints are kept because:

- The workflow trigger is the right place for flow control at the workflow level.
- The internal guard ensures `tag-commit` is safe to call from any workflow
  without accidentally tagging a non-main commit.

## Implementation Todos

**Already done on branch `4-version-and-tag-on-main-commit`:**

- [x] Add `dotnet` input to `steps/setup/action.yml` (default `'true'`; conditions
  `Setup .NET`, `Cache NuGet packages`, `Restore dependencies`)
- [x] Create `.github/workflows/main.yml` with `push`, `pull_request` and
  `workflow_dispatch` triggers; split into `version` (`contents: read`) and
  `tag` (`contents: write`, push-to-main only) jobs
- [x] Replace `GitVersion.yml` with `workflow: GitHubFlow/v1` (single line).
  The current committed file is a 113-line manual expansion of this preset;
  the single-line form is cleaner and unambiguously correct.

**Still required:**

- [ ] Commit all changes and push to the branch.
- [ ] Open a PR and verify the workflow runs successfully on the PR (no tag
  created; GitVersion computes a version without error).
- [ ] Merge the PR and verify the workflow runs on `main` and creates the tag.
- [ ] Confirm `git submodule status` shows the tag in a consuming repository.
- [ ] Close [issue #4](https://github.com/JohnLudlow/actions/issues/4) once verified.
