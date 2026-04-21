# Setup step (`actions/steps/setup`)

Common setup for jobs (checkout, versioning, .NET SDK, and restore).

## What it does

- Checks out the repository (LFS enabled, full history).
- Installs and runs GitVersion to compute version metadata.
- Exports version environment variables (`AssemblyVersion`, `AssemblyFileVersion`).
- Installs the requested .NET SDK.
- Caches NuGet packages.
- Runs `dotnet restore`.

## Inputs

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `dotnet-version` | No | `10.0.x` | Version passed to `actions/setup-dotnet` |

## Outputs

| Output | Meaning |
| --- | --- |
| `assemblySemVer` | Assembly semantic version from GitVersion |
| `assemblySemFileVer` | Assembly file version from GitVersion |
| `GitVersion_FullSemVer` | Full GitVersion semver |
| `semVer` | Semantic version |

## Notes

- This step includes `actions/checkout`. If your workflow already checks out the repo, you may want to remove one of them to avoid duplicate work.
