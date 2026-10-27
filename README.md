# platform-workflows (personal)

Reusable GitHub Actions workflows called by repos generated from `sampgreenwell-cyber/Backstage-Templates`. Mirrors the work setup (`essilor-ditac/platform-workflows`).

| Workflow | Used by | What it does |
|---|---|---|
| `container-app.yaml` | `dotnet-service` template | Build the Dockerfile, push to GHCR, `terraform apply` the repo's `infra/` (module `terraform-modules//container-app-service`). Auth: repo secrets from `set-platform-secrets.ps1`. |
| `destroy-container-app.yaml` | `dotnet-service` template | `terraform destroy` of that service, after the person running it types the repo name. |
| `container-app-v2.yaml` | `dotnet-service` template (new services) | Same as `container-app.yaml`, but no repo secrets: each job logs in to Vault with GitHub OIDC and reads the pipeline settings from `kv/platform/pipeline`. The deploy job's Azure credential is a new one per run from `azure/creds/pipeline` (Vault's Azure secrets engine), revoked when the job ends. Needs repo variable `VAULT_ADDR`; passes `vault_addr` to `infra/`. |
| `destroy-container-app-v2.yaml` | `dotnet-service` template (new services) | `destroy-container-app.yaml` with the same Vault login and per-run Azure credential. |
| `bicep-container-app.yaml` | `spring-container-app-lab` template | The same file as the work golden path: Maven build/test, Snyk, one image to ACR, Bicep deploy to dev, promotion to qa after approval. Auth: OIDC with GitHub environment variables. |
| `bicep-static-web-app.yaml` | `nextjs-web-app-lab` template | The same file as the work golden path: npm build/test, Snyk, Bicep (Static Web App + its APIM subscription to the BFF), build with `.env.<env>` and upload with a deploy token read at deploy time, then check `/api/health` and `/api/bff/health`. Auth: OIDC. |

Both Bicep pipelines take `release-model`: `trunk` (default; one branch, dev then qa after approval) or `environment-branches` (push to `dev` deploys dev, push to `qa` deploys qa after approval, only with code that already went through dev).

Each service's short caller pins `@v1`:
```yaml
jobs:
  deploy:
    uses: sampgreenwell-cyber/platform-workflows/.github/workflows/container-app.yaml@v1
    secrets: inherit
```
Inside a called workflow, `github.*` refers to the calling repo, so checkout, the repo name and `infra/` all come from the service.

New services use the Vault-backed pipeline instead, with no repo secrets:
```yaml
permissions:
  contents: read
  packages: write
  id-token: write
jobs:
  deploy:
    uses: sampgreenwell-cyber/platform-workflows/.github/workflows/container-app-v2.yaml@v1
```
Vault (`sampgreenwell-cyber/vault`, role `platform-pipeline`) trusts only the `-v2` files at `refs/tags/v1`, so a service repo can't read the pipeline secrets with a workflow of its own. Renaming or moving those files means updating the role's `job_workflow_ref` in that repo's `config/`.

## Access
- If this repo is **public**, any repo can call it; no setting needed.
- If it is **private**: Settings > Actions > General > Access > "Accessible from repositories owned by the user 'sampgreenwell-cyber'".

## Versioning
`v1` works like a subscription channel.
- **Non-breaking change** (works for every existing caller): merge to `main`, then move the tag:
  ```powershell
  git checkout main
  git pull
  git tag -f v1
  git push -f origin v1
  ```
- **Breaking change:** prefer a new file next to the old one (as with `container-app-v2.yaml`), released on `v1` like any other change: existing callers never reference it, so it's non-breaking for them. Update the templates' caller files for new services. Tag `v2` only for a change that has to replace every workflow at once, since moving to a new tag freezes the repos still on `v1`.

Organize by deployment pattern, not language: many templates, few workflows.
