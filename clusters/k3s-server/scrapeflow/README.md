# ScrapeFlow — Deployment Runbook

## Prerequisites

Before pushing this directory to the GitOps repo, create the following secrets manually on the cluster.
These are **never stored in git** — only `secretKeyRef` references appear in the manifests.

---

## 1. Create Secrets

> **Note:** The `scrapeflow` namespace is managed by `namespace.yaml` and created automatically by Flux.
> However, secrets must exist before the HelmReleases reconcile, so bootstrap the namespace manually first:
> ```bash
> kubectl create namespace scrapeflow
> ```
> Then create the secrets below, then push to git.

### Database credentials

```bash
kubectl create secret generic scrapeflow-db-credentials \
  --namespace scrapeflow \
  --from-literal=postgres-password=<strong-password> \
  --from-literal=postgres-user=scrapeflow \
  --from-literal=postgres-db=scrapeflow \
  --from-literal=database-url="postgresql+asyncpg://scrapeflow:<strong-password>@scrapeflow-postgresql:5432/scrapeflow"
```

### MinIO credentials

```bash
kubectl create secret generic scrapeflow-minio-credentials \
  --namespace scrapeflow \
  --from-literal=root-user=scrapeflow \
  --from-literal=root-password=<strong-password>
```

### App secrets

```bash
# Generate a Fernet key for LLM_KEY_ENCRYPTION_KEY:
# python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"

kubectl create secret generic scrapeflow-app-secrets \
  --namespace scrapeflow \
  --from-literal=clerk-secret-key=<clerk-secret-key> \
  --from-literal=llm-key-encryption-key=<fernet-key>
```

---

## 3. GitHub Actions secrets (ScrapeFlow repo)

Add these in the ScrapeFlow GitHub repo under **Settings → Secrets → Actions**:

| Secret | Value |
|--------|-------|
| `DOCKER_USERNAME` | DockerHub username (`k4rth`) |
| `DOCKER_PASSWD` | DockerHub access token (not account password) |

---

## 4. Push and reconcile

```bash
git add . && git commit -m "chore: add scrapeflow deployment" && git push
flux reconcile kustomization flux-system --with-source
flux get kustomizations -A
flux get helmreleases -A -n scrapeflow
```

---

## Service DNS (in-cluster)

| Service    | Hostname                          |
|------------|-----------------------------------|
| PostgreSQL | `scrapeflow-postgresql:5432`      |
| Redis      | `scrapeflow-redis-master:6379`    |
| MinIO      | `scrapeflow-minio:9000`           |
| NATS       | `scrapeflow-nats:4222`            |
