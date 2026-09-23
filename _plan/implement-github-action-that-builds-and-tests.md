# Add GitHub Actions CI workflow (build + test)

## Context

The repo is a minimal ASP.NET Core Web API (.NET 10) POC with no CI configured yet — no `.github` directory exists. The user wants a GitHub Action that builds the project and runs tests on every push/PR, so regressions are caught automatically instead of relying on manual `dotnet build`/`dotnet run` locally. There are currently no test projects in the solution (confirmed via search — no `*.Tests.csproj`, no xunit/nunit/mstest references), so the "test" step will be a safe no-op today (`dotnet test` with zero test projects exits 0) and will start doing real work automatically the moment a test project is added later — no workflow changes needed at that point.

## File to create

`.github/workflows/ci.yml`

## Workflow design

- **Triggers**: `push` to `main` and `pull_request` targeting `main`. Restricting `push` to `main` (rather than all branches) avoids duplicate runs when a feature branch is pushed and then immediately opened as a PR — the PR event covers feature-branch validation, push-to-main covers post-merge validation.
- **Runner**: `ubuntu-latest`. The Dockerfile/publish flow targets Windows containers, but that's irrelevant here — compiling and testing .NET code doesn't need Docker or Windows, and Ubuntu runners are faster/cheaper.
- **SDK version**: `10.0.x` via `actions/setup-dotnet@v4`. No `global.json` pins a version, so a floating minor (`10.0.x`) tracks SDK patch updates while matching the project's `TargetFramework net10.0`.
- **Caching**: use `actions/setup-dotnet`'s built-in NuGet cache (`cache: true`, `cache-dependency-path: '**/*.csproj'`) rather than a separate `actions/cache` step — simplest option for a one-project repo.
- **Steps**: separate Restore / Build / Test steps (using `--no-restore` / `--no-build` to avoid redundant work) for clearer failure attribution in the Actions UI, mirroring the documented `dotnet build ActionPoc.slnx` command from CLAUDE.md. `--configuration Release` throughout, consistent with the documented publish command.
- No `concurrency`/`cancel-in-progress` settings — unnecessary for a single small project with sub-minute build times.

## YAML content

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    name: Build and test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET SDK
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'
          cache: true
          cache-dependency-path: '**/*.csproj'

      - name: Restore
        run: dotnet restore ActionPoc.slnx

      - name: Build
        run: dotnet build ActionPoc.slnx --configuration Release --no-restore

      - name: Test
        run: dotnet test ActionPoc.slnx --configuration Release --no-build
```

## Verification

1. Locally run the same three commands from the repo root to confirm they succeed exactly as the workflow will run them:
   - `dotnet restore ActionPoc.slnx`
   - `dotnet build ActionPoc.slnx --configuration Release --no-restore`
   - `dotnet test ActionPoc.slnx --configuration Release --no-build` (should exit 0 with "no test projects found" since none exist yet)
2. Push the branch / open a PR against `main` and confirm in the GitHub Actions tab that the `CI` workflow triggers, all three steps (Restore/Build/Test) go green.
3. Once a real test project is added later, re-run step 2 to confirm `dotnet test` executes and reports actual results — validating the workflow's forward compatibility without further changes.
