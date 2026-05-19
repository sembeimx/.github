# sembeimx/.github

Centralized GitHub configuration for the **Sembei** organization. This repository holds:

- `profile/README.md` — public organization profile shown at [github.com/sembeimx](https://github.com/sembeimx)
- `.github/workflows/*.yml` — **reusable workflows** consumed by other Sembei repositories

---

## Reusable Workflows

### `deploy.yml` — Deploy to Sembei Platform

Builds a Docker image, pushes to Artifact Registry, and deploys to a Sembei VM (staging or production) via IAP SSH.

**Architecture it assumes**:

```
GitHub Actions ──── Workload Identity Federation ───▶ Service Account (per project + env)
                                                              │
                                                              ├─▶ Artifact Registry (push image)
                                                              │
                                                              └─▶ IAP SSH ─▶ VM ─▶ docker compose pull + migrate + up
```

**Security**:
- No long-lived service account keys — authentication via WIF (OIDC)
- Each call MUST declare `environment:` — WIF binding scopes by `(repository, environment)` pair, so a workflow without an environment cannot authenticate
- WIF provider restricted to `repository_owner == 'sembeimx'`
- Per-VM IAP tunnel access and OS Login privileges

**Usage example** — staging deploy on push to main:

```yaml
# .github/workflows/deploy-staging.yml in your project repo
name: Deploy staging
on:
  push:
    branches: [main]

jobs:
  deploy:
    uses: sembeimx/.github/.github/workflows/deploy.yml@main
    with:
      project_name: hda
      environment: staging
      vm_name: sembei-staging-vm
      vm_path: /opt/hablemosdeazucar/app
      image_repository: us-central1-docker.pkg.dev/sembei-websites/nori/hda
      service_account: github-hda-staging-deploy@sembei-websites.iam.gserviceaccount.com
      workload_identity_provider: projects/450896011520/locations/global/workloadIdentityPools/github-actions/providers/github
      health_check_host: hda-staging.sembei.mx
```

**Usage example** — production deploy on tag `v*`:

```yaml
# .github/workflows/deploy-production.yml
name: Deploy production
on:
  push:
    tags: ['v*']

jobs:
  deploy:
    uses: sembeimx/.github/.github/workflows/deploy.yml@main
    with:
      project_name: hda
      environment: production
      vm_name: sembei-vm
      vm_path: /opt/hablemosdeazucar/app
      image_repository: us-central1-docker.pkg.dev/sembei-websites/nori/hda
      service_account: github-hda-prod-deploy@sembei-websites.iam.gserviceaccount.com
      workload_identity_provider: projects/450896011520/locations/global/workloadIdentityPools/github-actions/providers/github
      health_check_host: hablemosdeazucar.com
```

**Per-project prerequisites** (one-time setup):

1. Two GCP Service Accounts created: `github-<project>-staging-deploy` and `github-<project>-prod-deploy`
2. SAs granted `artifactregistry.writer` on the `nori` Artifact Registry repository
3. SAs granted `iap.tunnelResourceAccessor` (project-level with IAM Condition scoping to the specific VM) and `compute.osLogin` on the target VM
4. WIF bindings: `principalSet://.../attribute.repo_env/sembeimx/<repo>:staging` and `:production`
5. Compose files committed to the project repo: `docker-compose.staging.yml` and `docker-compose.production.yml`
6. `.env.staging` and `.env.production` materialized on the VM under the project's `vm_path`

---

## Contributing

Changes to reusable workflows here affect ALL Sembei projects that reference them. Test changes by:

1. Pointing one project's `uses: sembeimx/.github/.github/workflows/deploy.yml@<your-branch>` to a feature branch
2. Trigger a deploy
3. Verify behavior
4. Merge to `main` once validated
