---
title: Tag on Main Workflow
status: proposed
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

## Proposed Solution

Add a GitHub Actions workflow file at `.github/workflows/tag-on-main.yml` that:

1. Triggers on every `push` to `main`
2. Checks out the repository with the options required by the `tag-commit` step
3. Installs and runs GitVersion directly to compute the semantic version
4. Runs `./steps/git/tag-commit` to apply and push the version tag

`./steps/setup` is intentionally **not used** in this workflow. That action
unconditionally runs `dotnet restore`, which fails in this repository because
there are no `.csproj` or `.sln` files. Only the GitVersion portion of the
setup logic is needed here.

A `GitVersion.yml` configuration file is also required. GitVersion has no prior
tags or configuration to derive a version from; without explicit configuration
the first-run output is undefined.

### Files to Create

| File | Notes |
| --- | --- |
| `.github/workflows/tag-on-main.yml` | New workflow file. The `.github/` directory does not yet exist and must be created. |
| `GitVersion.yml` | GitVersion configuration. Required for reproducible, predictable versioning. |

No existing files need to be modified.

## Workflow File

The workflow should be created with the following content:

```yaml
name: Tag on Main

on:
  push:
    branches:
      - main

jobs:
  tag:
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

      - name: Install GitVersion
        uses: gittools/actions/gitversion/setup@v4.2.0
        with:
          versionSpec: '6.4.x'

      - name: Determine Version
        id: gitversion
        uses: gittools/actions/gitversion/execute@v4.2.0

      - name: Tag Commit
        uses: ./steps/git/tag-commit
        with:
          version: ${{ steps.gitversion.outputs.GitVersion_FullSemVer }}
```

## GitVersion Configuration File

A `GitVersion.yml` must be created at the repository root. Without it,
GitVersion applies built-in defaults with no guarantee of producing a
meaningful or stable version number on a repository that has no prior tags.

The recommended starting configuration uses Mainline Development mode, which
increments the patch segment on every commit to `main` by default, and
respects conventional commit-style bump messages for minor and major
increments:

```yaml
mode: Mainline
major-version-bump-message: '\+semver:\s?(breaking|major)'
minor-version-bump-message: '\+semver:\s?(feature|minor)'
patch-version-bump-message: '\+semver:\s?(fix|patch)'
```

This will produce `1.0.0` on the first tagged run if the repository currently
has no tags, incrementing from there on subsequent commits.

## Design Notes

### Why `./steps/setup` is not used here

`./steps/setup` was designed for workflows that build .NET projects. It
unconditionally runs `dotnet restore`, which fails when no `.csproj` or `.sln`
file exists in the working directory. This repository contains only YAML
action definitions and Markdown documentation.

Rather than use `./steps/setup` and absorb unnecessary .NET SDK installation
and a failing restore step, this workflow invokes the GitVersion actions
directly — matching the same versions already pinned in `./steps/setup`.

### Why `permissions` is scoped to the job

`permissions: contents: write` is placed inside `jobs.tag`, not at the
workflow level. This follows the principle of least privilege: permissions
apply only to the job that requires them, rather than to every job in the
file. As additional jobs are added to this workflow in future, they will
default to read-only tokens.

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

| Option | Value | Reason |
| --- | --- | --- |
| `persist-credentials` | `true` | Explicit for clarity; this is the `actions/checkout` default. Ensures `git push` can authenticate. |
| `lfs` | `true` | Not currently used in this repo; included as a forward-compatible precaution per `tag-commit` docs. |
| `submodules` | `recursive` | This repo has no submodules; option is inert but included per `tag-commit` docs. |
| `fetch-depth` | `0` | Fetches full history. Required by GitVersion to compute the correct version. |

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

- [ ] Create `GitVersion.yml` at the repository root with the configuration
  shown above.
- [ ] Create the `.github/workflows/` directory structure.
- [ ] Create `.github/workflows/tag-on-main.yml` using the YAML shown above.
- [ ] Verify the workflow runs successfully on the next push to `main`.
- [ ] Confirm `git submodule status` shows the tag in a consuming repository.
- [ ] Close [issue #4](https://github.com/JohnLudlow/actions/issues/4) once verified.
