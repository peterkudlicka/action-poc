# Add CodeQL and secret-scanning to CI (enterprise-level security scanning)

## Context

The repo already has `.github/workflows/ci.yml` performing dotnet restore/build/test for the .NET 10 ASP.NET Core Web API solution (`ActionPoc.slnx` → `WebAPI/WebAPI.csproj`). The user now wants "enterprise level" security tooling layered on top: static application security analysis (CodeQL) and secret scanning, so vulnerabilities and accidentally-committed credentials are caught automatically in CI, not just correctness/build issues. `WebAPI/appsettings.json` already carries Azure AD tenant/client identifiers, so this is exactly the kind of repo where a future contributor could accidentally commit a real client secret or connection string — secret scanning has concrete value here, not just box-checking.

GitHub remote is `github.com/peterkudlicka/action-poc` (personal repo, visibility unconfirmed) — this matters because CodeQL's Security-tab (code scanning) integration is free for public repos but requires GitHub Advanced Security (paid) for private repos. The plan proceeds assuming the repo is public or will be; if it's private, CodeQL will still run and can be told to fail the build on findings even without Advanced Security, but SARIF upload to the Security tab won't work without GHAS.

## File to create

`.github/workflows/security.yml` (new file; `ci.yml` is left untouched — it continues to own build/test only).

One workflow, two independent jobs, sharing triggers: this keeps "scan the repo for problems" logically separate from "build and test the app," while avoiding a third file for a POC-sized repo.

## Design decisions

- **CodeQL**: separate job (not folded into `ci.yml`) because it needs its own `permissions: security-events: write` and a weekly `schedule` trigger that don't belong on the build/test job's minimal permissions.
  - `build-mode: manual` with explicit `dotnet restore`/`dotnet build` steps (mirroring `ci.yml`) rather than `autobuild` — more predictable for an SDK-style single-project solution, and keeps CodeQL's build aligned with the real build (same SDK pin, same restore).
  - Triggers: push to `main`, PR to `main`, plus a weekly cron (`0 6 * * 1`, Monday 06:00 UTC) — standard CodeQL default-setup cadence so drift/new CodeQL query packs get re-run even without code changes.
- **Secret scanning**: `gitleaks/gitleaks-action` as the second job. It's the de-facto standard OSS tool, ships a broad default ruleset for cloud/API keys, and needs only the built-in `GITHUB_TOKEN` for a personal-account repo (no `GITLEAKS_LICENSE` needed — that's only required for GitHub *organization*-owned repos). Checkout uses `fetch-depth: 0` so it scans full history, not just the diff.
  - GitHub's native secret scanning + push protection (Settings → Code security) is a repo *setting*, not a workflow — recommend the user enable it manually as a complementary, zero-YAML layer (free for public repos).
- Both jobs run on `ubuntu-latest`.

## YAML content

```yaml
name: Security

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'   # weekly, Monday 06:00 UTC

jobs:
  codeql:
    name: CodeQL analysis
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
      actions: read
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET SDK
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'
          cache: true
          cache-dependency-path: '**/*.csproj'

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: csharp
          build-mode: manual

      - name: Restore
        run: dotnet restore ActionPoc.slnx

      - name: Build
        run: dotnet build ActionPoc.slnx --configuration Release --no-restore

      - name: Perform CodeQL analysis
        uses: github/codeql-action/analyze@v3
        with:
          category: '/language:csharp'

  gitleaks:
    name: Secret scanning (gitleaks)
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Not doing now (flagged for later, out of scope for this pass)

- `.github/dependabot.yml` (NuGet ecosystem, weekly) for automated dependency-update PRs.
- `actions/dependency-review-action` as a PR-only job to block newly introduced vulnerable/incompatible NuGet packages.
- Enabling native GitHub secret scanning + push protection via repo Settings (manual, zero-YAML step the user should do themselves).

## Verification

1. Push `security.yml` on a branch and open a PR to `main` — confirm both `codeql` and `gitleaks` jobs appear in the Actions tab and pass.
2. After merge to `main`, check **Security → Code scanning alerts** for CodeQL results (may take a few minutes after `analyze` completes).
3. Optionally add `workflow_dispatch` temporarily to trigger on demand instead of waiting for the Monday cron.
4. To sanity-check gitleaks actually detects something, temporarily commit a dummy fake secret (e.g. an `AKIA`-prefixed 20-char string) on a scratch branch/PR, confirm the `gitleaks` job fails/annotates it, then revert the commit.
