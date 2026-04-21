# .NET Build step (`actions/steps/dotnet/build`)

Builds the repository and uploads build outputs as an artifact for downstream jobs.

## What it does

- Runs `dotnet build --no-restore` with the given configuration.
- Uploads `**/bin/**` and `**/obj/**` as the `build-artifacts` artifact.

## Inputs

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `configuration` | No | `Release` | Build configuration passed to `dotnet build` |

## Outputs / Artifacts

- Uploads artifact `build-artifacts` containing `bin/` and `obj/` folders.

## Notes

- Intended to be used after the [Setup step](./setup-step.md) (which restores packages).
