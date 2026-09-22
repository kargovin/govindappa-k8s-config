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
PG_PASS="your-password-here"

kubectl create secret generic scrapeflow-db-credentials \
  --namespace scrapeflow \
  --from-literal=postgres-password="$PG_PASS" \
  --from-literal=postgres-user=scrapeflow \
  --from-literal=postgres-db=scrapeflow \
  --from-literal=database-url="postgresql+asyncpg://scrapeflow:${PG_PASS}@scrapeflow-postgresql:5432/scrapeflow"
```

### Temporal database credentials

Temporal gets its **own Postgres instance** (`infrastructure/temporal-postgres.yaml`), not databases
on the app one — its schema is owned by `temporal-sql-tool`, not Alembic, and its data is in-flight
work (ScrapeFlow ADR-009 §2a). The Secret holds only user and password; the database names
(`temporal`, `temporal_visibility`) are fixed in the manifest because the Temporal server is
configured with them by name.

```bash
TEMPORAL_PG_PASS="your-password-here"

kubectl create secret generic scrapeflow-temporal-db-credentials \
  --namespace scrapeflow \
  --from-literal=postgres-user=temporal \
  --from-literal=postgres-password="$TEMPORAL_PG_PASS"
```

> ⚠️ **Create this Secret before pushing the manifest.** Flux applies within a minute; without it
> the pod sits in `CreateContainerConfigError` (harmless, recovers once the Secret exists).
>
> ⚠️ **Both databases are created on first boot only**, by the postgres image's entrypoint against an
> empty `PGDATA`. If the pod ever starts with the init ConfigMap missing, `temporal_visibility` is
> never created and mounting it later does nothing. The PVC outlives the StatefulSet, so the reset is
> explicit: `kubectl -n scrapeflow delete pvc data-scrapeflow-temporal-postgresql-0`, then let the
> StatefulSet recreate the pod.
>
> Verify: `kubectl -n scrapeflow exec sts/scrapeflow-temporal-postgresql -- psql -U temporal -l`
> lists both databases; each has 0 tables until the Temporal server's schema init container runs.

### Temporal server

`infrastructure/temporal.yaml` — no Secret of its own; it reads `scrapeflow-temporal-db-credentials`
above. One `temporalio/server` process (all four services, SQL visibility) behind ClusterIP
`scrapeflow-temporal:7233`. A `schema` **init container** (`temporalio/admin-tools`, **same tag as
the server — bump both lines together**) runs `temporal-sql-tool setup-schema`/`update-schema` on
every pod start; it is a no-op once the databases are at the image's version, which is what makes a
tag bump safe. The manifest is `strategy: Recreate` so the old server is gone before the new pod
migrates the schema.

> ⚠️ **`NUM_HISTORY_SHARDS` (4) is immutable after the first start** — it is persisted in
> `cluster_metadata_info`. Changing it means wiping the Temporal database (PVC delete above).
>
> The `temporal` CLI ships in admin-tools, not the server image. Health check from a one-off pod:
>
> ```bash
> kubectl -n scrapeflow run tcli --restart=Never --image=temporalio/admin-tools:1.31.0 --command -- \
>   temporal operator cluster health --address scrapeflow-temporal:7233
> kubectl -n scrapeflow logs tcli      # SERVING
> kubectl -n scrapeflow delete pod tcli
> ```
>
> (`kubectl run --rm -i` loses the output to the pod's own teardown.)
>
> Verify after a bump: `kubectl -n scrapeflow logs deploy/scrapeflow-temporal -c schema` — on an
> up-to-date database it reads `found zero updates from current version <n>` for both databases.
>
> ⚠️ **Do not `kubectl rollout restart` a Flux-managed Deployment** — Flux strips the `restartedAt`
> annotation on its next reconcile, which rolls the Deployment a second time. Use
> `kubectl -n scrapeflow delete pod -l app=scrapeflow-temporal`.
>
> ⚠️ **A replacement pod may exit once with `failed to start ringpop`** (exit 1) and come back on the
> kubelet's restart: it bootstraps the ring from `cluster_membership`, which still lists the previous
> pod's IP. Self-healing, ~90 s. Only investigate if it loops.
> The dynamic config (`scrapeflow-temporal-dynamicconfig`) is re-read live; editing it needs no restart.

### MinIO credentials

> **Note:** The MinIO official chart requires keys named `rootUser` and `rootPassword` (not `root-user`/`root-password`).

```bash
MINIO_PASS="your-password-here"

kubectl create secret generic scrapeflow-minio-credentials \
  --namespace scrapeflow \
  --from-literal=rootUser=scrapeflow \
  --from-literal=rootPassword="$MINIO_PASS"
```

### App secrets

`scrapeflow-app-secrets` holds **three** keys:

| Key | Purpose | Rotatable? |
|-----|---------|------------|
| `clerk-secret-key` | Clerk backend secret (`sk_test_…` dev / `sk_live_…` prod) — JWT verification + Clerk API calls | Yes — see "Rotating the Clerk secret" below |
| `llm-key-encryption-key` | Fernet key — encrypts users' stored **LLM API keys** at rest (Postgres) | **No** — rotating orphans all existing ciphertext |
| `credentials-encryption-key` | Fernet key — encrypts stored **proxy URLs / cookies** at rest (Postgres) | **No** — rotating orphans all existing ciphertext |

> The two Fernet keys must be **distinct** values, each generated with:
> ```bash
> python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
> ```
> Once data has been encrypted with them, **never regenerate them** — the ciphertext becomes permanently undecryptable and stored LLM keys / proxy creds are lost.

```bash
CLERK_SECRET="your-clerk-secret-key"          # sk_test_… (dev) or sk_live_… (prod)
LLM_FERNET_KEY="your-llm-fernet-key"          # Fernet key #1
CREDS_FERNET_KEY="your-credentials-fernet-key" # Fernet key #2 (must differ from #1)

kubectl create secret generic scrapeflow-app-secrets \
  --namespace scrapeflow \
  --from-literal=clerk-secret-key="$CLERK_SECRET" \
  --from-literal=llm-key-encryption-key="$LLM_FERNET_KEY" \
  --from-literal=credentials-encryption-key="$CREDS_FERNET_KEY"
```

#### Rotating the Clerk secret (e.g. dev → production instance)

Only `clerk-secret-key` changes when moving Clerk instances — **do not** re-create the secret
from scratch (that would force re-supplying the two Fernet keys, which must not change). Patch just
the one key, then restart the API to pick it up:

```bash
NEW_SK="sk_live_..."   # from the Clerk production instance

kubectl patch secret scrapeflow-app-secrets -n scrapeflow \
  --type merge \
  -p "{\"data\":{\"clerk-secret-key\":\"$(printf %s "$NEW_SK" | base64 -w0)\"}}"

kubectl rollout restart deployment/scrapeflow-api -n scrapeflow
```

> The frontend's Clerk **publishable** key is baked into the API image at build time
> (`VITE_CLERK_PUBLISHABLE_KEY` build-arg, sourced from the GitHub Actions secret below), **not**
> from this Kubernetes secret. Swapping instances therefore also requires updating that GitHub secret
> and rebuilding/redeploying the `scrapeflow-api` image — the `pk_live_…` and `sk_live_…` must come
> from the same Clerk instance or every request 401s. `CLERK_AUTHORIZED_PARTIES` in `app/api.yaml`
> is already the prod origin (`https://scrapeflow.govindappa.com`) and needs no change.

---

## 3. GitHub Actions secrets (ScrapeFlow repo)

Add these in the ScrapeFlow GitHub repo under **Settings → Secrets → Actions**:

| Secret | Value |
|--------|-------|
| `DOCKER_USERNAME` | DockerHub username (`k4rth`) |
| `DOCKER_PASSWD` | DockerHub access token (not account password) |
| `VITE_CLERK_PUBLISHABLE_KEY` | Clerk **publishable** key (`pk_test_…` dev / `pk_live_…` prod) — baked into the frontend bundle at api-image build time. Must match the instance of `clerk-secret-key` above. |

> **Note:** the api image only rebuilds on changes under `api/**` or `frontend/**`. After changing
> `VITE_CLERK_PUBLISHABLE_KEY` alone, trigger a rebuild via **Actions → Build & Push → Run workflow**
> (`workflow_dispatch`) so the new key gets baked in.

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
| Temporal PG| `scrapeflow-temporal-postgresql:5432` |
| Temporal   | `scrapeflow-temporal:7233`        |
