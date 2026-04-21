# Git tag commit step (`actions/steps/git/tag-commit`)

Tags the `main` branch commit with a version tag on push.

## What it does

- On `push` to `main`, runs `git tag` and pushes the tag back to the repository.

## Inputs

None.

## Notes / caveats

- This action relies on GitVersion output being available in the job to compute the tag name.
- The action performs `git push` and requires appropriate permissions/token configuration in the calling workflow.
