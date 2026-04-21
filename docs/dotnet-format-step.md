# .NET Format step (`actions/steps/dotnet/format`)

Runs `dotnet format` in verification mode.

## What it does

- Checks out the repo.
- Installs .NET SDK.
- Caches NuGet packages.
- Runs `dotnet format --verify-no-changes`.

## Inputs

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `dotnet-version` | No | `10.0.x` | Version passed to `actions/setup-dotnet` |

## Notes

- This step is currently commented out in the main workflow (`.github/workflows/main.yml`).
