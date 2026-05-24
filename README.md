# Performance Enabler frontend template

This repository is a **GitHub template** for Performance Enabler Digital Platform client apps. Do not commit changes here directly — clone it, customise locally, and use the **"Use this template"** button to generate per-client repositories.

Repos generated from this template are populated by the platform's `provisionAppIntegrated` Function (or by the self-service provisioning script for pull-request / export-only modes). The CI workflow expects six GitHub Action secrets to be set on the generated repository:

| Secret | Source |
|---|---|
| `AZURE_STATIC_WEB_APPS_API_TOKEN` | Auto-set by `az staticwebapp create` (or by `provisionAppIntegrated`) |
| `VITE_DATAVERSE_URL` | The client's app-data Dataverse URL |
| `VITE_TENANT_ID` | The client's Entra tenant ID |
| `VITE_SP_CLIENT_ID` | The client's published-app Entra App Registration client ID |
| `VITE_REDIRECT_URI` | The SWA's `https://*.azurestaticapps.net` URL |
| `VITE_APP_SLUG` | The app's slug (matches a `*.app.json` file in `public/`) |

For more, see [platform docs](https://github.com/Simply-Now-Enabler/) (operator-internal).

## Build script — load-bearing exit code

The workflow at `.github/workflows/azure-static-web-apps.yml` runs an explicit `npm ci && npm run build` step (via `actions/setup-node@v4`) **before** the `Azure/static-web-apps-deploy@v1` action. The SWA action is configured with `skip_app_build: true` so Oryx is not involved in the build at all — it only handles the upload. The `build` script in `package.json` is `tsc --noEmit && vite build`: if `tsc --noEmit` encounters any TypeScript error, the script exits non-zero, the "Install and build" workflow step fails red, and the SWA action never runs. No stale or broken content is deployed.

An earlier iteration of this workflow relied on Oryx invoking `npm run build` via the `app_build_command` parameter. That approach is insufficient: Oryx catches the script's non-zero exit and falls through to "Oryx was unable to determine the build steps. Continuing assuming the assets in this folder are already built" — reporting "Build Complete" with an empty workspace. This Oryx silent-fallback was observed live on 2026-05-23 via planted-type-error dogfooding with all three prior A5 fixes in place. The explicit pre-build step + `skip_app_build: true` bypasses Oryx's build phase entirely and restores standard GitHub Actions shell semantics. The `app_build_command` parameter is retained on the SWA action as defence-in-depth: if `skip_app_build` is ever accidentally removed, it prevents Oryx from using its own framework detection.

Do not remove the "Install and build" step, remove `skip_app_build: true`, or replace the build script with a tolerant pattern such as `vite build` alone — any of these re-introduces the silent-fallback failure mode. Client repos provisioned from this template inherit the workflow and script at provision time; preserving this behaviour in downstream repos is the operator's responsibility.
