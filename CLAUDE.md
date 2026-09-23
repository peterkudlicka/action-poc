# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

ActionPoc is a minimal ASP.NET Core Web API (.NET 10) with Azure AD (Microsoft Entra ID) authentication wired in via Microsoft.Identity.Web. It currently exposes a single scaffolded `WeatherForecastController` endpoint and serves as a starting point/proof of concept.

## Commands

Run all commands from the repository root unless noted.

- Restore/build: `dotnet build ActionPoc.slnx`
- Run locally (HTTPS profile, port 7009 / HTTP 5215): `dotnet run --project WebAPI`
- Publish: `dotnet publish WebAPI/WebAPI.csproj -c Release`
- Build the Docker image: `docker build -f WebAPI/Dockerfile .` (context must be the repo root, since the Dockerfile references `WebAPI/WebAPI.csproj` relative to root)

There are no test projects in the solution yet.

## Architecture

- `ActionPoc.slnx` is the (new-format) solution file referencing the single `WebAPI` project.
- `WebAPI/Program.cs` is the composition root: registers Microsoft Identity Web JWT bearer authentication (`AddMicrosoftIdentityWebApi`) bound to the `AzureAd` config section, adds controllers and OpenAPI, and wires the standard middleware pipeline (HTTPS redirection → authentication → authorization → controllers).
- Authentication/authorization is Azure AD-backed: every controller is expected to carry `[Authorize]` and `[RequiredScope(RequiredScopesConfigurationKey = "AzureAd:Scopes")]` (see `WeatherForecastController`), enforcing the scope declared in `AzureAd:Scopes` in configuration.
- `AzureAd` settings (tenant, client ID, scopes) live in `WebAPI/appsettings.json`. `appsettings.Development.json` only overrides logging.
- OpenAPI is enabled only in the Development environment (`app.MapOpenApi()` gated on `IsDevelopment()`).
- The project targets Windows containers by default (`DockerDefaultTargetOS=Windows`); `WebAPI/Dockerfile` uses nanoserver base images and expects to be built with the repo root as the Docker context.
