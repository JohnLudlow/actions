# CodeQL step (`actions/steps/codeql`)

Runs GitHub CodeQL analysis for the C# codebase.

## What it does

- Checks out the repo (full history, LFS).
- Initializes CodeQL for `csharp`.
- Installs .NET SDK.
- Caches NuGet packages.
- Runs `dotnet restore` and `dotnet build`.
- Uploads results via `github/codeql-action/analyze`.

## Inputs

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `dotnet-version` | No | `10.0.x` | Version passed to `actions/setup-dotnet` |
| `configuration` | No | `Release` | Build configuration passed to `dotnet build` |

## Notes

- The workflow must grant `security-events: write` permissions for CodeQL upload (see `.github/workflows/main.yml`).
