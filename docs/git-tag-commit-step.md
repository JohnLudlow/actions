# Git tag commit step (`actions/steps/git/tag-commit`)

Tags the `main` branch commit with a version tag on push.

## What it does

- On `push` to `main`, runs `git tag` and pushes the tag back to the repository.

## Inputs

None.

## Checkout requirements

This action runs `git tag` and `git push` against the repository, so the calling workflow **must** check out the repository before invoking this action. The checkout must be configured with the following options:

| Option | Value | Reason |
| --- | --- | --- |
| `persist-credentials` | `true` | Required so that `git push origin <tag>` can authenticate using the token supplied by the checkout action. |
| `lfs` | `true` | Ensures binary files stored via Git LFS are available and the LFS pointer metadata is intact. |
| `submodules` | `recursive` | Ensures any reusable actions referenced as submodules are fully initialised. |
| `fetch-depth` | `0` | Fetches the full commit history, which is required by GitVersion to compute the correct version number. |

Example:

```yml
- uses: actions/checkout@v5
  with:
    persist-credentials: true
    lfs: true
    submodules: recursive
    fetch-depth: 0
```

## Notes / caveats

- This action relies on GitVersion output being available in the job to compute the tag name.
- The action performs `git push` and requires appropriate permissions/token configuration in the calling workflow.
