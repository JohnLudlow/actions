# .NET Test step (`actions/steps/dotnet/test`)

Runs unit tests, produces TRX test results, and generates/publishes a coverage summary.

## What it does

- Downloads `build-artifacts`.
- Runs `dotnet test --no-build` with XPlat Code Coverage enabled.
- Publishes TRX results via `dorny/test-reporter`.
- Generates a Markdown coverage summary using ReportGenerator.
- Uploads the rendered coverage report as an artifact.
- On `main` pushes, uploads a `coverage-baseline` artifact (for delta comparisons on PRs).

## Inputs

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `configuration` | No | `Release` | Build configuration passed to `dotnet test` |
| `assembly-semver` | Yes | (none) | Used as the coverage report title |
| `gitversion-full-semver` | Yes | (none) | Used in coverage artifact name |

## Required permissions

- Uses `gh api` to download baseline artifacts on PRs; the calling workflow should grant `actions: read` if needed.

## Notes

- This step expects the workflow to have run the [Setup step](./setup-step.md) and [Build step](./dotnet-build-step.md) first.
