# Add Azure App Service deploy workflow (code deploy)

## Context

The repo already has `ci.yml` (build/test), `security.yml` (CodeQL + gitleaks), and `sbom.yml` (SBOM generation), but nothing that actually ships the app anywhere. The user wants to deploy `WebAPI` to an Azure App Service using a "code deploy" (`dotnet publish` output), not a container — even though the repo also has a Windows-container `Dockerfile` (`WebAPI/Dockerfile`, `DockerDefaultTargetOS=Windows` in the csproj), that path is intentionally out of scope here.

Decisions made with the user before writing this:
- **Auth:** OIDC federated credentials via `azure/login@v2` — no stored password/secret, consistent with the user's stated concern (from the same session) that credentials should never be committed or stored as long-lived secrets if avoidable.
- **Target App Service:** assumed to already exist / will be created manually by the user outside this repo — no Bicep/ARM/Terraform in scope (confirmed via repo-wide search: no existing IaC anywhere).
- **Trigger:** `workflow_dispatch` only for now — no auto-deploy on push to `main`, since this is still a POC without a staging gate.

## File to create

`.github/workflows/deploy.yml` (new, dedicated file).

## Design decisions

- **Match existing conventions exactly**: `actions/checkout@v4`, `actions/setup-dotnet@v4` with `dotnet-version: '10.0.x'`, `cache: true`, `cache-dependency-path: '**/*.csproj'`, and restore against the **solution** `ActionPoc.slnx` (not the csproj directly) — same as `ci.yml` and `security.yml`.
- **Permissions**: `id-token: write` + `contents: read`. `id-token: write` is new to this repo — it's required for `azure/login@v2` to mint an OIDC token; no other workflow here needs it.
- **Publish, don't build+test again**: this workflow does its own `restore` → `publish` rather than depending on `ci.yml`'s build artifact, since there's no artifact-passing set up between workflows yet and `workflow_dispatch` is decoupled from `ci.yml`'s `push`/`pull_request` triggers. `dotnet publish` implies a build, so no separate `dotnet build` step is needed.
- **`--no-restore` on publish**: consistent with the `--no-restore`/`--no-build` chaining pattern already used in `ci.yml`.
- **Azure login via `azure/login@v2`**: takes `client-id`, `tenant-id`, `subscription-id` from GitHub **secrets** (`AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`). These are identifiers, not passwords — but they identify a specific federated-credential trust relationship, so they're still stored as secrets rather than variables, per Azure's own guidance for this pattern.
- **Deploy via `azure/webapps-deploy@v3`**: `app-name` comes from a GitHub **variable** (`vars.AZURE_WEBAPP_NAME`), not a secret, since a Web App name isn't sensitive. `package` points at the publish output directory.
- **Action version pins**: major-version tags (`@v4`, `@v2`, `@v3`), matching the repo's existing convention (`actions/checkout@v4`, `gitleaks/gitleaks-action@v2`, `github/codeql-action@v3`) rather than pinning to a full SHA. Dependabot's `github-actions` ecosystem entry will keep these patched.

## YAML content

```yaml
name: Deploy

on:
  workflow_dispatch:

jobs:
  deploy:
    name: Deploy to Azure App Service
    runs-on: ubuntu-latest
    permissions:
      id-token: write
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

      - name: Publish
        run: dotnet publish WebAPI/WebAPI.csproj -c Release -o ${{ github.workspace }}/publish --no-restore

      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Azure App Service
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ vars.AZURE_WEBAPP_NAME }}
          package: ${{ github.workspace }}/publish
```

## Prerequisites (outside this repo — not automated by this workflow)

- Create/reuse an Azure AD App Registration and add a **federated credential** scoped to this GitHub repo (entity type: branch `main`, or a GitHub Environment if you want an approval gate later) — this is what lets `azure/login@v2` authenticate with no stored secret.
- Grant that App Registration's service principal the **Website Contributor** (or `Contributor`) role on the target App Service / resource group.
- Add GitHub repo **secrets**: `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`.
- Add GitHub repo **variable**: `AZURE_WEBAPP_NAME`.
- Verify the target App Service's runtime stack actually offers **.NET 10** (very recently GA) — if Azure's built-in stack list doesn't have it yet, revisit with a self-contained publish as a fallback.

## Note (not acted on)

`WebAPI/appsettings.json` still has a dummy `ConnectionStrings:DefaultConnection` value planted earlier to test `security.yml`'s gitleaks job. Once a real connection string is needed for the deployed app, it belongs in **App Service Application Settings** (or a Key Vault reference), not `appsettings.json` — flagging so it isn't forgotten, not part of this change.

## Verification

1. Complete the prerequisites above (Azure federated credential + role assignment, GitHub secrets/variable).
2. Trigger manually: `gh workflow run deploy.yml` (or **Actions → Deploy → Run workflow** in the GitHub UI), then `gh run watch` to confirm the `deploy` job succeeds.
3. Hit the deployed app: `curl -i https://<app-name>.azurewebsites.net/weatherforecast` — expect **HTTP 401** (the controller is `[Authorize]`-protected), which confirms the app started correctly and the auth middleware is active.
