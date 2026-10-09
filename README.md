# platform-workflows (personal)

Reusable GitHub Actions workflows called by repos generated from `sampgreenwell-cyber/Backstage-Templates`. Mirrors the work setup (`essilor-ditac/platform-workflows`).

| Workflow | Used by | What it does |
|---|---|---|
| `container-app.yaml` | `dotnet-service` template | Build the Dockerfile, push to GHCR, `terraform apply` the repo's `infra/` (module `terraform-modules//container-app-service`). Auth: repo secrets from `set-platform-secrets.ps1`. |
| `destroy-container-app.yaml` | `dotnet-service` template | `terraform destroy` of that service, after the person running it types the repo name. |
| `bicep-container-app.yaml` | `spring-container-app-lab` template | The same file as the work golden path: Maven build/test, Snyk, one image to ACR, Bicep deploy to dev, promotion to qa after approval. Auth: OIDC with GitHub environment variables. |

Each service's short caller pins `@v1`:
```yaml
jobs:
  deploy:
    uses: sampgreenwell-cyber/platform-workflows/.github/workflows/container-app.yaml@v1
    secrets: inherit
```
Inside a called workflow, `github.*` refers to the calling repo, so checkout, the repo name and `infra/` all come from the service.

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
- **Breaking change:** tag `v2`; callers move only when their file says `@v2`. Update the templates' caller files for new services.

Organize by deployment pattern, not language: many templates, few workflows.
