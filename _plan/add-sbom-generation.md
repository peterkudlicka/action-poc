# Add SBOM generation via anchore/sbom-action

## Context

The repo already has `ci.yml` (build/test), `security.yml` (CodeQL + gitleaks), and `dependabot.yml` (dependency-update PRs). The user now wants a Software Bill of Materials generated as a downloadable workflow artifact, using `anchore/sbom-action` (Syft-based), to give visibility into exactly which NuGet packages (including transitive dependencies) ship in the app — useful for compliance/supply-chain audits.

## File to create

`.github/workflows/sbom.yml` (new, dedicated file — not added to `ci.yml` or `security.yml`).

**Why a new file**: `ci.yml` is purely build/test; `security.yml`'s existing jobs (CodeQL, gitleaks) share the pattern "scan for problems and report findings." SBOM generation is different in kind — it produces a compliance/inventory artifact rather than scanning for problems — so it gets its own file rather than blurring either existing workflow's responsibility. The cost is one small extra file, negligible for this repo size.

## Design decisions

- **Scan accuracy for .NET**: Syft's `.NET` dependency detection reads `obj/project.assets.json` (produced by `dotnet restore`), which reflects the fully-resolved transitive dependency tree — scanning raw `.csproj` files only shows direct top-level version ranges. So the job restores first (`dotnet restore ActionPoc.slnx`), then points `anchore/sbom-action`'s `path:` at the `WebAPI` project directory (not repo root, not `WebAPI/obj` directly — Syft auto-discovers `obj/project.assets.json` under the given path).
- **Output format**: `spdx-json` only. SPDX is the NTIA-minimum-conformant, industry/GitHub-preferred format; no current consumer needs CycloneDX too, so skip doubling the output.
- **Artifact upload**: rely on the action's built-in `upload-artifact: true` (default) rather than a separate `actions/upload-artifact` step. Set an explicit `artifact-name` and `upload-artifact-retention: 30` (days) — shorter than the 90-day default, appropriate for a POC.
- **Triggers**: push to `main` only, plus `workflow_dispatch` for on-demand runs. Not on every PR — a fresh SBOM per open PR is noisy/low-value while review is still in progress; the SBOM only needs to reflect what's actually merged. No weekly cron either (unlike `security.yml`) — SBOM content only changes when dependencies change, i.e., on merge, not on a schedule.
- **Permissions**: `contents: read` only — no `security-events: write` needed since this doesn't submit to GitHub's dependency-submission API or Security tab (out of scope here).
- **Action version pin**: `anchore/sbom-action@v0`, consistent with this repo's existing convention of pinning to a major-version tag (`actions/checkout@v4`, `github/codeql-action@v3`, `gitleaks/gitleaks-action@v2`) rather than a full SHA. Dependabot's `github-actions` ecosystem entry (already in `.github/dependabot.yml`) will keep it patched.

## YAML content

```yaml
name: SBOM

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  sbom:
    name: Generate SBOM
    runs-on: ubuntu-latest
    permissions:
      contents: read

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

      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          path: WebAPI
          format: spdx-json
          artifact-name: sbom-webapi.spdx.json
          upload-artifact: true
          upload-artifact-retention: 30
```

## Verification

1. Push to `main` (or trigger manually via **Actions → SBOM → Run workflow**).
2. Open the completed run in the **Actions** tab; confirm the **Artifacts** section lists `sbom-webapi.spdx.json`, downloadable as a zip.
3. Download and unzip; open the `.spdx.json` file and confirm:
   - `packages[]` contains an entry for `Microsoft.Identity.Web` with a real resolved version (not just a version range).
   - A transitive-only package (something pulled in by `Microsoft.Identity.Web` but not directly referenced in `WebAPI.csproj`) is also present — confirms `obj/project.assets.json`-based resolution rather than shallow `.csproj` parsing.
