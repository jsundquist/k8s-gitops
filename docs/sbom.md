# SBOM dashboard deployment

The Next.js SBOM dashboard runs at `sbom.digital-portfolio.net`. Its source, Dockerfile and image workflow live in the `sbom` repository; this repository holds the manifests. The sbom repo's `docs/kubernetes.md` is the handoff these manifests follow.

## Workloads

One image (`ghcr.io/jsundquist/sbom`) runs three ways, all in the `sbom` namespace:

| Name | Kind | Sync wave | Notes |
|---|---|---|---|
| `postgres` | StatefulSet, 5Gi PVC (k3s local-path) | 0 | `postgres:16`, non-root, read-only rootfs |
| `migrate` | Job, Argo CD `Sync` hook | 1 | `DATABASE_URL` only; reruns on every sync |
| `web` | Deployment, 1 replica | 2 | Service `web`, Ingress `sbom`, probes on `/api/livez` and `/api/healthz` |
| `worker` | Deployment, 1 replica, `Recreate` | 2 | No port, no probes |

## Decisions

- **Migrate is a Sync hook, not PreSync.** A PreSync hook runs before anything else in the sync, including Postgres and the secrets, so on the first sync it would have nothing to connect to. As a Sync-phase hook in wave 1 it runs after the database is ready and before web and worker roll out.
- **Node directly, not npm.** `npm` runs a shell that swallows SIGTERM, so graceful shutdown never ran. Web uses the image default and the worker sets `command`.
- **Single replica for web.** The rate limiter is in memory per pod. Web is cut off on rollout because `next start` does not drain; accepted.
- **Worker uses `Recreate`.** Two workers must not enrich at once, even briefly.
- **Postgres in-cluster, no backups yet.** The data is rebuildable from re-uploads, but a backup job is a stage 9 item.
- **Public image.** Same reasoning as the blog: no pull secret. Make the GHCR package public by hand after the first push.
- **Hardening.** Non-root, read-only root filesystem, no capabilities, `RuntimeDefault` seccomp, and `emptyDir`s for the few writable paths. This mirrors the compose file.
- **Staging issuer first** on the Ingress, then `letsencrypt-prod`, like the blog.

## Secrets

Two Sealed Secrets, committed encrypted. Namespace and name are part of the seal, so they must match exactly.

| Secret | Keys | Used by |
|---|---|---|
| `sbom-db` | `postgres-password`, `database-url` | postgres, migrate, web, worker |
| `sbom-app` | `NEXTAUTH_SECRET`, `GITHUB_CLIENT_SECRET`, `UPLOAD_TOKENS` | web |

Non-secret settings (`NEXTAUTH_URL`, `TRUST_PROXY`, `OWNER_GITHUB_ID`, `GITHUB_CLIENT_ID`, ...) are in `apps/sbom/configmap.yaml`.

### Sealing them

Run from the repo root with the droplet's kubeconfig. Nothing below is written to disk unencrypted.

```sh
export KUBECONFIG=~/.kube/k8s-droplet.yaml

PW=$(openssl rand -hex 24)
kubectl create secret generic sbom-db -n sbom --dry-run=client -o yaml \
  --from-literal=postgres-password="$PW" \
  --from-literal=database-url="postgres://sbom:$PW@postgres:5432/sbom" \
  | kubeseal --format yaml > apps/sbom/sealed-db.yaml

read -rs GH_SECRET   # paste the OAuth app's client secret, then Enter
UPLOAD=$(openssl rand -hex 24)
kubectl create secret generic sbom-app -n sbom --dry-run=client -o yaml \
  --from-literal=NEXTAUTH_SECRET="$(openssl rand -hex 32)" \
  --from-literal=GITHUB_CLIENT_SECRET="$GH_SECRET" \
  --from-literal=UPLOAD_TOKENS="$UPLOAD" \
  | kubeseal --format yaml > apps/sbom/sealed-app.yaml
```

`$UPLOAD` is the bearer token the upload action (`action.yml` in the sbom repo) needs. Save it in a password manager before closing the shell, because the sealed file cannot be decrypted locally. Then uncomment `sealed-db.yaml` and `sealed-app.yaml` in `apps/sbom/kustomization.yaml`.

## First deploy checklist

1. Sealed Secrets controller running (`argocd/droplet/apps/sealed-secrets.yaml`, chart 2.22.0 from `bitnami.github.io/sealed-secrets`), sealing key backed up outside git:
   `kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml` into a password manager, never into git.
2. `A` record for `sbom` pointing at the droplet (done).
3. First image pushed from the sbom repo's `main`, GHCR package set to Public, tag copied into `apps/sbom/kustomization.yaml`.
4. GitHub OAuth app created: homepage `https://sbom.digital-portfolio.net`, callback `https://sbom.digital-portfolio.net/api/auth/callback/github`. Its client id and your numeric GitHub user id go in `configmap.yaml`.
5. Secrets sealed and uncommented as above.
6. Commit and push. The `sbom` Application is picked up by the root and syncs by wave.

## Follow-ups

- Verify the client IP seen by the app, with a forged `X-Forwarded-For`. The rate limiter trusts the leftmost value, so Traefik must overwrite it and k3s ServiceLB must not mask the real address.
- Switch the Ingress to `letsencrypt-prod` once the staging certificate issues.
- Tag-bump job in the sbom repo, as the blog has.
- Postgres backups, NetworkPolicies, and a pinned Postgres digest (stage 9).
