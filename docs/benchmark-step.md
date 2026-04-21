# Benchmark step (`actions/steps/benchmark`)

Runs a BenchmarkDotNet benchmark project and (optionally) compares against a baseline captured from the `main` branch.

## What it does

- Downloads build artifacts produced by the build job.
- On `push` to `main`, runs benchmarks and uploads the `BenchmarkDotNet.Artifacts` folder as an Actions artifact (the baseline).
- On `pull_request`, downloads the most recent baseline artifact (if present) and runs benchmarks against it.
- Appends any markdown results in `<artifacts>/results/*.md` to the GitHub Actions job summary.

## Inputs

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `configuration` | No | `Release` | Build configuration passed to `dotnet run` |
| `project` | Yes | (none) | Path to the benchmark project `.csproj` |
| `job` | No | `short` | BenchmarkDotNet job preset (e.g. `short`, `medium`, `long`, `dry`, `default`) |
| `filter` | No | `*` | BenchmarkDotNet filter pattern |
| `artifacts` | No | `BenchmarkDotNet.Artifacts` | BenchmarkDotNet artifacts output folder |
| `exporters` | No | `GitHub` | BenchmarkDotNet exporters (typically `GitHub`) |
| `baselineArtifactName` | No | `benchmark-baseline` | Artifact name used to store the baseline |
| `baselineDir` | No | `BenchmarkBaseline` | Directory to extract the baseline artifact into |

## Example usage (workflow)

```yml
benchmark:
  needs: build
  runs-on: windows-latest
  env:
    CONFIGURATION: Release
  permissions:
    contents: read
    issues: write
    actions: read

  steps:
  - uses: actions/checkout@v5
    with:
      fetch-depth: 0
      submodules: recursive

  - uses: ./actions/steps/setup

  - uses: ./actions/steps/benchmark
    with:
      configuration: ${{ env.CONFIGURATION }}
      project: src/GameEngineAdapter.Benchmarks/GameEngineAdapter.Benchmarks.csproj
      job: short
      exporters: GitHub
      filter: "*"
```

## Notes

- The benchmark project must be an executable (`OutputType=Exe`) that uses BenchmarkDotNet (for example by using `BenchmarkSwitcher` in `Program.cs`).
- The action passes BenchmarkDotNet CLI arguments via `dotnet run -- ...`.
- Baseline download uses `gh api` to list repository artifacts; ensure the job has `actions: read` permissions.
