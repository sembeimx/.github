# Sembei Platform — Migration Playbook

How to onboard a new project (or migrate an existing one) onto the Sembei platform's GitHub Actions deploy flow.

This document is the operational complement to [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml). HDA was the pilot — replicate this for `sembei`, `ocampo`, and future clients.

---

## Prerequisites (one-time per platform, already done)

- ✅ Workload Identity Pool `github-actions` in GCP project `sembei-websites`
- ✅ OIDC Provider `github` with `repository_owner == 'sembeimx'` condition
- ✅ Artifact Registry `nori` repo (`us-central1-docker.pkg.dev/sembei-websites/nori`)
- ✅ `sembei-staging-vm` and `sembei-prod-vm` (prod) provisioned with Docker + Caddy + Redis
- ✅ Cloud SQL `sembei-mysql` (shared instance)
- ✅ Reusable workflow at `sembeimx/.github/.github/workflows/deploy.yml@main`

---

## Per-project migration steps

Replace `<project>` (e.g. `sembei`, `ocampo`) and `<client-domain>` (e.g. `clientdomain.com`) as you go.

### 1. Cloud SQL (DB + user)

```bash
# Create database
gcloud sql databases create <project>_staging --instance=sembei-mysql --project=sembei-websites
gcloud sql databases create <project>_production --instance=sembei-mysql --project=sembei-websites

# Create users with passwords from 1P (vault Administracion → item "Stagings" for staging;
# create vault item "<PROJECT> — Production (Cloud SQL + .env)" for prod)
gcloud sql users create <project>_staging --host='%' --instance=sembei-mysql --password=<from-1P>
gcloud sql users create <project>_user_prod --host='%' --instance=sembei-mysql --password=<from-1P>

# Scope privileges (run from any VM that can reach 10.229.0.3)
mysql -h 10.229.0.3 -u root <<SQL
GRANT ALL PRIVILEGES ON <project>_staging.* TO '<project>_staging'@'%';
GRANT ALL PRIVILEGES ON <project>_production.* TO '<project>_user_prod'@'%';
ALTER USER '<project>_staging'@'%' WITH MAX_USER_CONNECTIONS 5;
FLUSH PRIVILEGES;
SQL
```

If migrating from an existing DB (legacy MySQL or in-container), do an import:

```bash
gcloud storage cp legacy-dump.sql.gz gs://<project>-staging-uploads/db-snapshots/
gcloud sql import sql sembei-mysql \
  gs://<project>-staging-uploads/db-snapshots/legacy-dump.sql.gz \
  --database=<project>_staging --project=sembei-websites
```

### 2. GCP IAM (per-project service accounts)

```bash
PROJECT_NUM=$(gcloud projects describe sembei-websites --format='value(projectNumber)')

# Service accounts (per env)
for ENV in staging prod; do
  gcloud iam service-accounts create github-<project>-${ENV}-deploy \
    --display-name="GitHub Actions: <PROJECT> ${ENV} deploy" \
    --project=sembei-websites
done

# Artifact Registry writer (both envs push to same repo, workflow controls tags)
for ENV in staging prod; do
  gcloud artifacts repositories add-iam-policy-binding nori \
    --location=us-central1 \
    --member="serviceAccount:github-<project>-${ENV}-deploy@sembei-websites.iam.gserviceaccount.com" \
    --role="roles/artifactregistry.writer" \
    --project=sembei-websites
done

# IAP tunnel (no condition — project-wide is acceptable since condition-based
# scoping with iap.googleapis.com/TunnelInstance is fragile; OS Login per-VM
# already enforces which VMs the SA can SSH to)
for ENV in staging prod; do
  gcloud projects add-iam-policy-binding sembei-websites \
    --member="serviceAccount:github-<project>-${ENV}-deploy@sembei-websites.iam.gserviceaccount.com" \
    --role="roles/iap.tunnelResourceAccessor" \
    --condition=None
done

# Custom role for gcloud compute ssh metadata lookup (create once, reuse)
# gcloud iam roles create gheActionsIapSshLookup --project=sembei-websites \
#   --permissions=compute.projects.get,compute.instances.get --stage=GA
for ENV in staging prod; do
  gcloud projects add-iam-policy-binding sembei-websites \
    --member="serviceAccount:github-<project>-${ENV}-deploy@sembei-websites.iam.gserviceaccount.com" \
    --role="projects/sembei-websites/roles/gheActionsIapSshLookup" \
    --condition=None
done

# OS Admin Login per-VM (sudo on the target VM)
gcloud compute instances add-iam-policy-binding sembei-staging-vm \
  --zone=us-central1-a \
  --member="serviceAccount:github-<project>-staging-deploy@sembei-websites.iam.gserviceaccount.com" \
  --role="roles/compute.osAdminLogin" \
  --project=sembei-websites

gcloud compute instances add-iam-policy-binding sembei-prod-vm \
  --zone=us-central1-a \
  --member="serviceAccount:github-<project>-prod-deploy@sembei-websites.iam.gserviceaccount.com" \
  --role="roles/compute.osAdminLogin" \
  --project=sembei-websites

# actAs on each VM's attached SA
gcloud iam service-accounts add-iam-policy-binding sembei-staging-vm-sa@sembei-websites.iam.gserviceaccount.com \
  --member="serviceAccount:github-<project>-staging-deploy@sembei-websites.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser" \
  --project=sembei-websites

gcloud iam service-accounts add-iam-policy-binding sembei-prod-vm@sembei-websites.iam.gserviceaccount.com \
  --member="serviceAccount:github-<project>-prod-deploy@sembei-websites.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser" \
  --project=sembei-websites

# WIF binding (scoped by repo + environment)
for ENV in staging production; do
  SA_ENV=$([ "$ENV" = "production" ] && echo "prod" || echo "$ENV")
  gcloud iam service-accounts add-iam-policy-binding \
    github-<project>-${SA_ENV}-deploy@sembei-websites.iam.gserviceaccount.com \
    --role="roles/iam.workloadIdentityUser" \
    --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUM}/locations/global/workloadIdentityPools/github-actions/attribute.repo_env/sembeimx/<repo>:${ENV}" \
    --project=sembei-websites
done
```

### 3. DNS (Cloud DNS in project `sembei-1753378396720`)

```bash
# Staging subdomain → staging-vm IP
gcloud dns record-sets create <project>-staging.sembei.mx. \
  --zone=sembei-mx --project=sembei-1753378396720 \
  --type=A --ttl=300 --rrdatas=34.173.239.53

# Production (if migrating client domain, point A record to sembei-prod-vm)
gcloud dns record-sets create <client-domain>. \
  --zone=<client-domain-zone> --project=sembei-1753378396720 \
  --type=A --ttl=300 --rrdatas=35.222.32.133
```

### 4. Provision repo on each VM

For each VM (`sembei-staging-vm` for staging container, `sembei-prod-vm` for prod container):

```bash
gcloud compute ssh <vm-name> --zone=us-central1-a --project=sembei-websites --tunnel-through-iap

# As root on VM:
sudo -u sembei mkdir -p /opt/<project>
sudo chown sembei:sembei /opt/<project>

# Generate dedicated deploy key for sembei
sudo -u sembei ssh-keygen -t ed25519 -N "" -C "<vm-name>-<project>" -f /home/sembei/.ssh/id_ed25519_<project>

# Append to SSH config
sudo -u sembei tee -a /home/sembei/.ssh/config <<EOF

Host github-<project>
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_<project>
  IdentitiesOnly yes
EOF

# Display pub key (add to GitHub repo as deploy key, read-only)
sudo -u sembei cat /home/sembei/.ssh/id_ed25519_<project>.pub

# After adding deploy key on GitHub:
sudo -u sembei git clone git@github-<project>:sembeimx/<repo>.git /opt/<project>/app

# Materialize .env from 1P (one-time, then deploys handle code)
sudo -u sembei tee /opt/<project>/app/<env_file_path> > /dev/null <<EOF
# ... values from 1P ...
EOF
sudo chmod 600 /opt/<project>/app/<env_file_path>
```

### 5. Configure docker auth for `sembei` user (each VM)

```bash
sudo -u sembei gcloud auth configure-docker us-central1-docker.pkg.dev --quiet
```

### 6. Grant `artifactregistry.reader` to each VM's attached SA

```bash
# Staging-vm SA
gcloud artifacts repositories add-iam-policy-binding nori \
  --location=us-central1 \
  --member="serviceAccount:sembei-staging-vm-sa@sembei-websites.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.reader" --project=sembei-websites

# Prod-vm SA (sembei-prod-vm@…)
gcloud artifacts repositories add-iam-policy-binding nori \
  --location=us-central1 \
  --member="serviceAccount:sembei-prod-vm@sembei-websites.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.reader" --project=sembei-websites
```

(Both should already be granted after HDA migration, but verify.)

### 7. Caddyfile entries

Edit `/opt/sembei/caddy/Caddyfile` on each VM and add:

```caddy
# On sembei-staging-vm — basic_auth protects from public eyes
<project>-staging.sembei.mx {
	@protected not path /health
	basic_auth @protected {
		staging $2a$14$<bcrypt-hash>
	}
	reverse_proxy <project>-staging:8000
	encode gzip zstd
}

# On sembei-prod-vm — no basic_auth (public production)
<client-domain> {
	reverse_proxy <project>-production:8000
	encode gzip zstd
}
```

Reload:

```bash
sudo docker compose -f /opt/sembei/docker-compose.yml exec caddy caddy reload --config /etc/caddy/Caddyfile
```

### 8. Compose files in the project repo

`docker-compose.staging.yml` and `docker-compose.production.yml` should pull from AR (NOT build on-server):

```yaml
name: <project>-staging  # or -production

services:
  app:
    image: us-central1-docker.pkg.dev/sembei-websites/nori/<project>:staging
    container_name: <project>-staging  # or -production
    restart: unless-stopped
    pull_policy: always
    command: ["gunicorn", "asgi:app", "-c", "gunicorn.conf.py", "--chdir", "rootsystem/application", "--forwarded-allow-ips=*"]
    env_file:
      - <path-to-.env.staging>
    networks:
      - sembei_default
    labels:
      com.sembei.client: "<project>"
      com.sembei.env: "staging"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

networks:
  sembei_default:
    external: true
```

### 9. Workflow shims in `.github/workflows/`

`deploy-staging.yml`:

```yaml
name: Deploy staging
on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    permissions:
      contents: read
      id-token: write
    uses: sembeimx/.github/.github/workflows/deploy.yml@main
    with:
      project_name: <project>
      environment: staging
      vm_name: sembei-staging-vm
      vm_path: /opt/<project>/app
      image_repository: us-central1-docker.pkg.dev/sembei-websites/nori/<project>
      service_account: github-<project>-staging-deploy@sembei-websites.iam.gserviceaccount.com
      workload_identity_provider: projects/450896011520/locations/global/workloadIdentityPools/github-actions/providers/github
      health_check_host: <project>-staging.sembei.mx
```

`deploy-production.yml`: same structure, swap `staging` → `production`, `vm_name: sembei-prod-vm`, and `health_check_host: <client-domain>`.

### 10. Initial deploy

```bash
git push origin main         # staging fires
# verify https://<project>-staging.sembei.mx works
git tag -a v0.1.0 -m "Initial Sembei platform deploy"
git push origin v0.1.0       # production fires
# verify https://<client-domain> works
```

### 11. Decommission old setup

After verification:
- Stop old containers on prod-vm: `docker stop <project>-staging <project>-db && docker rm <project>-staging <project>-db`
- Remove obsolete volume: `docker volume rm <project>_db_data` (if container-MySQL was used)
- Update prod-vm Caddyfile: redirect any old staging URL to new one (`<project>-staging.sembei.mx`)

---

## Common gotchas (learned from HDA migration)

| Symptom | Cause | Fix |
|---------|-------|-----|
| Workflow `startup_failure` | Caller workflow missing `permissions: id-token: write` | Add permissions block at job level in the shim |
| IAP error 4033 'not authorized' | IAM condition uses `compute.googleapis.com/Instance` type, IAP wants `iap.googleapis.com/TunnelInstance` | Drop the condition (project-wide is fine; OS Login per-VM enforces VM scope) |
| `Could not fetch resource: Required 'compute.projects.get'` | SA lacks project metadata read | Grant custom role `gheActionsIapSshLookup` (compute.projects.get + compute.instances.get) at project level |
| `iam.serviceAccounts.actAs` denied | SA can't impersonate the VM's attached SA | Grant `roles/iam.serviceAccountUser` on the VM's SA to the deploy SA |
| `fatal: detected dubious ownership` | Repo on VM owned by a user other than `sembei` | `chown -R sembei:sembei /opt/<project>` |
| `Could not resolve hostname github-<project>` | SSH config alias not in sembei's `.ssh/config` | Generate deploy key for sembei, add to repo, set up SSH config alias |
| `Unauthenticated request` on `docker pull` | VM's attached SA lacks AR reader OR docker not configured for user | Grant `roles/artifactregistry.reader` to VM SA + run `gcloud auth configure-docker` as the executing user |

---

## Verification checklist after migration

- [ ] `git push origin main` triggers staging deploy workflow successfully
- [ ] `https://<project>-staging.sembei.mx/health` returns 200 OK
- [ ] `git tag vX.Y.Z && git push origin vX.Y.Z` triggers production deploy successfully
- [ ] `https://<client-domain>/health` returns 200 OK
- [ ] Old containers removed from sembei-prod-vm
- [ ] Old DNS records (if any) redirect or cleaned up
- [ ] README of project documents the new flow with environment URLs
