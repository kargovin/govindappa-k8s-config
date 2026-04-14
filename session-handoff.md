# Session Handoff — 2026-04-14

## What was done

### Audit
Full review of all manifests. Key findings:

| # | Issue | Status |
|---|-------|--------|
| 1 | `nginx.ingress.kubernetes.io/from-to-www-redirect` annotation ignored by Traefik — www was being dual-served, not redirected | Fixed |
| 2 | Missing `external-dns.alpha.kubernetes.io/hostname` on `simple-app` and `jupyter` ingresses | Fixed (simple-app); jupyter still open |
| 3 | MLflow namespace `mlflow` has no `Namespace` manifest and no `createNamespace: true` on HelmRelease — must be manually pre-created on bootstrap | Open |
| 4 | `test-app-ingress` missing `ingressClassName: traefik` | Open |
| 5 | Jupyter using `:latest` image tag — not reproducible | Open |
| 6 | No resource requests/limits on `simple-app` nginx and `test-app` | Open |
| 7 | No liveness/readiness probes on `simple-app` nginx | Open |

### Fix applied — `clusters/k3s-server/simple-app.yaml` (commit `96314c7`)
- Added `Traefik Middleware/www-redirect` (RedirectRegex, permanent 301)
- Split single ingress into:
  - `static-web-ingress` → serves `govindappa.com` (canonical)
  - `www-redirect-ingress` → 301s `www.govindappa.com` → `govindappa.com`
- Added missing `external-dns.alpha.kubernetes.io/hostname` to both ingresses
- Removed no-op nginx annotation

### Verified on cluster
- Both ingresses live, both TLS certs issued and `Ready` by cert-manager
- Flux reconciled commit `96314c7` successfully on both kustomizations

## Open items for next session
1. Add `Namespace` manifest for `mlflow` (or `createNamespace: true` on HelmRelease)
2. Add `ingressClassName: traefik` to `test-app-ingress`
3. Pin Jupyter image to a specific tag
4. Add `external-dns` annotation to `jupyter-ingress`
5. Add resource limits + health probes to `simple-app` nginx
6. Add resource limits to `test-app`
7. Implement proper www redirect for `simple-app` — done, but consider whether SEO canonicalization (`<link rel="canonical">`) is also needed in the HTML
