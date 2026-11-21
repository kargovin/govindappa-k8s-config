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

### Temporal namespace

`app/temporal-init-job.yaml` — Job `scrapeflow-temporal-init` registers the Temporal namespace
`scrapeflow` (a row in the `temporal` database, not the k8s namespace) with **30-day retention**, or
resets the retention on an existing one — the manifest is the source of truth for it. No Secret; it
talks to `scrapeflow-temporal:7233`. Same admin-tools tag as the server.

> Verify: the tcli pod above with `temporal operator namespace describe --namespace scrapeflow
> --address scrapeflow-temporal:7233` → `WorkflowExecutionRetentionTtl 720h0m0s`.
>
> **Re-run it after the Temporal database is wiped** (or to re-apply retention):
> `kubectl -n scrapeflow delete job scrapeflow-temporal-init`; Flux recreates it on the next reconcile.

### Temporal Web UI

`infrastructure/temporal-ui.yaml` — `temporalio/ui` behind ClusterIP `scrapeflow-temporal-ui:8080`.
**No Ingress, deliberately:** the UI can cancel, terminate, signal and reset any workflow and has no
auth of its own, and every tenant's runs share the one `scrapeflow` namespace. Reach it with:

```bash
kubectl -n scrapeflow port-forward svc/scrapeflow-temporal-ui 8081:8080
# → http://localhost:8081 (opens on the scrapeflow namespace)
```

> The UI image has its own version line (2.x) — it is **not** bumped with the server/admin-tools
> tag. Local port 8081 because the app repo's local compose UI holds 8080; any free port works —
> reads and writes (CSRF-token protected) both verified on a port other than 8080.

### API

`app/api.yaml` — Deployment `scrapeflow-api`: **2 replicas, `RollingUpdate`** (`maxSurge: 1`,
`maxUnavailable: 0`) behind Service `scrapeflow-api` and the ingress. Each pod runs `alembic upgrade
head` on startup under a Postgres advisory lock, so pods that start together migrate one at a time
(a waiter logs `Another process holds the migration lock; waiting for it`).

> ⚠️ **During a rollout an old pod and a new pod serve side by side**, with the new pod's migration
> already applied. A migration that drops or renames a column the running code still reads must be
> split across two releases: stop reading it, then drop it.
>
> Each pod keeps its own WebSocket count, so `WS_MAX_CONNECTIONS_PER_USER` (25) is per pod — a user
> can hold up to 2 × that. Live status reaches every pod (`pg_notify` goes to each listener).
>
> `preStop` sleeps 5 s so Traefik stops routing to a pod before it gets SIGTERM.

### Workflow worker

`app/workflow-worker.yaml` — Deployment `scrapeflow-workflow-worker`: the **api image** (same
`ImagePolicy`, so it follows the api's tag) running `/app/.venv/bin/python -m app.workflows.worker_main`,
polling Temporal task queue `workflow`. No Service — it only dials out to `scrapeflow-temporal:7233`.
No new Secret: it reuses the DB, MinIO and app secrets.

> ⚠️ **`/app/.venv/bin/python`, never bare `python` and not `uv run`** — the image's system
> interpreter has none of the app's packages, and `uv run` as PID 1 ignores a SIGTERM that arrives
> during its own startup, so a stop then waits out the grace period and ends in a SIGKILL. The pod
> also needs the three Fernet keys and `PUBLIC_BASE_URL`: `app.settings` refuses to load without
> them (so does the cleanup CronJob, which runs the same image).
>
> `terminationGracePeriodSeconds: 30` must stay above the worker's 20 s drain on SIGTERM.
>
> Verify: `kubectl -n scrapeflow logs deploy/scrapeflow-workflow-worker` → `Workflow worker started
> … task_queue=workflow`; the Web UI's task queue `workflow` lists the poller.

### HTTP worker (Temporal)

`app/http-worker-temporal.yaml` — Deployment `scrapeflow-http-worker-temporal`: the **http-worker
image** (`ImagePolicy` `scrapeflow-http-worker-policy`) polling task
queue `scrape-http` for the `Scrape` activity. The image's only Deployment since H.1 removed the
NATS-mode one. No Service, no new Secret; **no NATS and no DB env** — Temporal + MinIO only.

> `terminationGracePeriodSeconds: 30` must stay above the worker's 20 s drain on SIGTERM.
>
> Verify: `kubectl -n scrapeflow logs deploy/scrapeflow-http-worker-temporal` → `Temporal worker
> started … task_queue=scrape-http`; the Web UI's task queue `scrape-http` lists the poller.

### LLM worker (Temporal)

`app/llm-worker-temporal.yaml` — Deployment `scrapeflow-llm-worker-temporal`: the **llm-worker
image** (`ImagePolicy` `scrapeflow-llm-worker-policy`) polling task
queues `llm` (`LLMExtract`) and `content` (Clean, Validate). The image's only Deployment since H.1
removed the NATS-mode one. No Service, no new Secret; **no NATS and no DB env** — Temporal + MinIO
+ the LLM-key Fernet key only.

> ⚠️ **`terminationGracePeriodSeconds: 420` must stay above the worker's 400 s drain on SIGTERM**
> (`LLM_GRACEFUL_SHUTDOWN_SECONDS`, sized to one warm-up + request). A rollout with a call in flight
> can take ~7 minutes; a shorter grace period cancels the call and the retry re-bills the user's key.
>
> Verify: `kubectl -n scrapeflow logs deploy/scrapeflow-llm-worker-temporal` →
> `temporal_worker_started … task_queue=llm`; the Web UI's task queue `llm` lists the poller.

### Playwright worker (Temporal)

`app/playwright-worker-temporal.yaml` — Deployment `scrapeflow-playwright-worker-temporal`: the
**playwright-worker image** (`ImagePolicy` `scrapeflow-playwright-worker-policy`) with
polling task queues `scrape-playwright` (`Scrape`) and `crawl` (the crawl's
robots/sitemap/link activities). Headed Chrome under Xvfb. The image's only Deployment since H.1
removed the NATS-mode one. No Service, no new Secret; **no NATS and no DB env** — Temporal + MinIO
+ the credentials Fernet key only.

> ⚠️ **`terminationGracePeriodSeconds: 660` must stay above the worker's 630 s drain on SIGTERM**
> (`PLAYWRIGHT_GRACEFUL_SHUTDOWN_SECONDS`, sized to the 300 s `timeout_seconds` maximum twice over +
> upload). A rollout with a render in flight can take ~11 minutes; an idle pod exits at once.
>
> The container's start is `entrypoint.sh`: Xvfb, then `exec python` as PID 1 — never `xvfb-run` as
> the entrypoint (a dead worker would look healthy).
>
> Verify: `kubectl -n scrapeflow logs deploy/scrapeflow-playwright-worker-temporal` →
> `temporal_worker_started … task_queue=scrape-playwright`; the Web UI's task queue
> `scrape-playwright` lists the poller.

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

### Destinations key

`scrapeflow-destination-secrets` holds one key, `destination-encryption-key`: the Fernet key that
encrypts users' saved **destination secrets** (bucket keys, logins, webhook signing secrets) in
Postgres. It is its own Secret because the delivery worker reads it and nothing else (ScrapeFlow
ADR-015 §4) — that pod must never reference `scrapeflow-app-secrets`. The api, the workflow worker
and the cleanup CronJob read it too: `app.settings` refuses to load without it, so **create it
before pushing any manifest that references it** (an api pod without it crash-loops; with
`maxUnavailable: 0` the old pods keep serving).

> **Not rotatable** — like the two keys above, regenerating it makes every saved destination's
> secret undecryptable. Generate it once:

```bash
DEST_FERNET_KEY="$(python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())")"

kubectl create secret generic scrapeflow-destination-secrets \
  --namespace scrapeflow \
  --from-literal=destination-encryption-key="$DEST_FERNET_KEY"
```

`PUBLIC_BASE_URL` (`https://scrapeflow.govindappa.com`) is a plain value in the manifests: its host
names email Message-IDs and it is the base of links in messages.

### Delivery worker's MinIO user

The delivery worker reads ScrapeFlow's own copies of results and nothing more: MinIO user
`scrapeflow-delivery` with policy `delivery-read` — `s3:GetObject` on `scrapeflow-results/*` and
`s3:GetBucketLocation` on the bucket; no list, no write, no delete. Both are declared in
`infrastructure/minio.yaml` (`policies`, `users`); the chart's post-install/post-upgrade Job
creates them. The password lives in its own Secret, which **must exist before the HelmRelease
upgrades** — the Job mounts it; without it the Job's pod cannot start and the upgrade fails.

```bash
DELIVERY_MINIO_PASS="$(openssl rand -hex 24)"

kubectl create secret generic scrapeflow-minio-delivery-credentials \
  --namespace scrapeflow \
  --from-literal=password="$DELIVERY_MINIO_PASS"
```

> ⚠️ **The chart creates a user once and never changes its password.** To rotate: patch the
> Secret, then `mc admin user add m scrapeflow-delivery <new password>` by hand (it replaces the
> password and keeps the policy — checked on compose's MinIO) and restart the delivery worker.
>
> ⚠️ **The chart's default `users` value is a `console` user, password `console123`, policy
> `consoleAdmin`** (chart 5.4.0's `values.yaml`). This HelmRelease never set `users` before, so that
> user was most likely created at install. Declaring `users` stops the chart re-creating it; it
> does **not** remove it. Check `mc admin user list` below; removing it is
> `mc admin user remove m console`.

Verify (port-forward + an `mc` container on your machine; the root password never leaves your shell):

```bash
kubectl -n scrapeflow port-forward svc/scrapeflow-minio 19000:9000 &
ROOT_PASS="$(kubectl -n scrapeflow get secret scrapeflow-minio-credentials -o jsonpath='{.data.rootPassword}' | base64 -d)"
DELIVERY_PASS="$(kubectl -n scrapeflow get secret scrapeflow-minio-delivery-credentials -o jsonpath='{.data.password}' | base64 -d)"
docker run --rm --network host -e ROOT_PASS -e DELIVERY_PASS --entrypoint sh minio/mc -c '
  mc alias set m http://127.0.0.1:19000 scrapeflow "$ROOT_PASS" >/dev/null
  mc admin user list m                       # scrapeflow-delivery enabled, delivery-read
  mc admin policy info m delivery-read
  P=scrapeflow-results/delivery-check/probe.txt
  echo probe | mc pipe "m/$P" >/dev/null
  mc alias set d http://127.0.0.1:19000 scrapeflow-delivery "$DELIVERY_PASS" >/dev/null
  mc cat "d/$P" >/dev/null && echo "read: ok"
  mc ls d/scrapeflow-results >/dev/null 2>&1 && echo "list: ALLOWED (wrong)" || echo "list: refused"
  echo x | mc pipe d/scrapeflow-results/delivery-check/write.txt >/dev/null 2>&1 && echo "write: ALLOWED (wrong)" || echo "write: refused"
  mc rm "d/$P" >/dev/null 2>&1 && echo "delete: ALLOWED (wrong)" || echo "delete: refused"
  mc rm --recursive --force m/scrapeflow-results/delivery-check/ >/dev/null'
kill %1
```

> Expected: `read: ok`, then `refused` three times (the script was run against compose's MinIO,
> which creates the same user and policy). The probe is written and removed with the root
> user under `delivery-check/`, so no user's object is touched.

### Delivery worker NetworkPolicy

`app/delivery-worker-networkpolicy.yaml` — `NetworkPolicy` `scrapeflow-delivery-worker`, selecting
`app: scrapeflow-delivery-worker` (ScrapeFlow ADR-015 §3–§4). **No ingress.** Egress: CoreDNS
(`kube-system`, `k8s-app: kube-dns`, 53 UDP/TCP); the Temporal pod on 7233; the MinIO pod
(`app: minio, release: scrapeflow-minio`) on 9000; and `0.0.0.0/0` on any port **except** the pod
and service CIDRs, RFC 1918, `100.64.0.0/10`, link-local, loopback and **the node's own address**.
IPv4 only: an IPv6 destination address is refused by the policy.

> ⚠️ **The node's address is hardcoded** (`139.99.121.44/32` — the node's InternalIP, which is its
> public IP; CoreDNS's `NodeHosts` lists the same). The code-level address check admits a public
> IP, and a pod reaching its own node arrives from inside the machine, where the host trusts it
> (ScrapeFlow BUG-032). If the node's address changes, or `kubectl get nodes -o wide` shows another
> node or address, add it to the `except` list.
>
> ⚠️ **Enforcement is unproven on this cluster**: k3s enforces NetworkPolicy through its embedded
> controller unless it was disabled. Until the delivery worker exists the policy selects no pod.
> The proof is ScrapeFlow's Release M gate, run from the delivery worker pod: TCP to
> `scrapeflow-postgresql:5432`, `scrapeflow-redis-master:6379`, the node's address and the node's
> cni0 address (`10.42.0.1` — the pod's default gateway) **fails**, to a public host **succeeds**.

Before the first push, confirm the values the policy assumes (read-only):

```bash
kubectl get nodes -o wide                                              # INTERNAL-IP / EXTERNAL-IP
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDRs}'; echo        # inside 10.42.0.0/16
kubectl get svc kubernetes -n default -o jsonpath='{.spec.clusterIP}'; echo   # 10.43.0.1
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl -n scrapeflow get pods -l app=scrapeflow-temporal
kubectl -n scrapeflow get pods -l app=minio,release=scrapeflow-minio
```

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
| Temporal PG| `scrapeflow-temporal-postgresql:5432` |
| Temporal   | `scrapeflow-temporal:7233`        |
| Temporal UI| `scrapeflow-temporal-ui:8080` (port-forward only) |
