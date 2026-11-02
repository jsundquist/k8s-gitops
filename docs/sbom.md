# SBOM dashboard deployment

The Next.js SBOM dashboard runs at `sbom.digital-portfolio.net`. Its source, Dockerfile and image workflow live in the `sbom` repository; this repository holds the manifests. The sbom repo's `docs/kubernetes.md` is the handoff these manifests follow.

## Workloads

Two images built from the same commit and tagged identically run in the `sbom` namespace: `ghcr.io/jsundquist/sbom` (web only, Next.js standalone) and `ghcr.io/jsundquist/sbom-worker` (worker and migrate; has `tsx`, `src/`, `scripts/` and `drizzle/`):

| Name | Kind | Sync wave | Notes |
|---|---|---|---|
| `postgres` | StatefulSet, 5Gi PVC (k3s local-path) | 0 | `postgres:16`, non-root, read-only rootfs |
| `migrate` | Job, Argo CD `Sync` hook | 1 | `DATABASE_URL` only; reruns on every sync |
| `web` | Deployment, 1 replica | 2 | Service `web`, Ingress `sbom`, probes on `/api/livez` and `/api/healthz` |
| `worker` | Deployment, 1 replica, `Recreate` | 2 | No port, no probes |

## Decisions

- **Migrate is a Sync hook, not PreSync.** A PreSync hook runs before anything else in the sync, including Postgres and the secrets, so on the first sync it would have nothing to connect to. As a Sync-phase hook in wave 1 it runs after the database is ready and before web and worker roll out.
- **Node directly, not npm.** `npm` runs a shell that swallows SIGTERM, so graceful shutdown never ran. Web uses the image default (`node server.js`); the worker sets `command` explicitly and migrate runs `scripts/migrate.ts`.
- **Single replica for web.** The rate limiter is in memory per pod. Web is cut off on rollout because Next.js does not drain; accepted.
- **Worker uses `Recreate`.** Two workers must not enrich at once, even briefly.
- **Postgres in-cluster, no backups yet.** The data is rebuildable from re-uploads, but a backup job is a stage 9 item.
- **Public images.** Same reasoning as the blog: no pull secret. Make both GHCR packages (`sbom` and `sbom-worker`) public by hand after the first push.
- **Two images, one tag.** Web needs no `tsx` or sources, so it ships separately. Migrate and worker must run from the `sbom-worker` image, which is why `kustomization.yaml` lists both. The release `bump` job sets both tags with `yq` (`images[0]` is web, the other is matched by name), so keep that order and never pin them by hand.
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

## Gotchas hit during the first deploy

- **A wrong OAuth client secret gives `OAuthCallback` at sign-in.** The pod log said `incorrect_client_credentials`. The sealed value was 18 characters instead of GitHub's 40, from a bad terminal paste (bracketed paste leaking into `read -rs`). Check the length (`echo ${#VAR}`) before sealing, and verify in the pod with `kubectl exec deploy/web -n sbom -- sh -c 'echo ${#GITHUB_CLIENT_SECRET}'`.
- **A changed Secret does not restart pods.** Environment variables are read at startup, so run `kubectl rollout restart deploy/web -n sbom` after resealing.
- **To change one sealed key without touching the rest**, use `kubeseal --merge-into apps/sbom/sealed-app.yaml`.
- **gitleaks flags Sealed Secrets ciphertext** as an API key, so `.gitleaks.toml` allow-lists it narrowly.

## Deploys

The sbom repository's `release.yml` builds, scans (Trivy) and pushes both images on every push to `main`. Its `bump` job then pins the new tag on both in `apps/sbom/kustomization.yaml` here (commit `chore(sbom): deploy <tag>`), and Argo CD syncs: the migrate Job runs first, then web and worker roll out. A separate `token-expiry` job warns 14 days before the token expires.

- **Token:** a fine-grained PAT, repository `jsundquist/k8s-gitops` only, `Contents: Read and write`, stored as the `GITOPS_TOKEN` secret in the sbom repository. It is separate from the blog's token so either can be revoked on its own.
- **Poll delay.** Argo CD polls git every few minutes and has no webhook (the API is not exposed), so a deploy can take several minutes to start. Hard-refresh the Application to start it sooner.
- **Image retention** is not implemented for this repository: it builds only on pushes to `main`, so images accumulate slowly.

## More notes

- Backups: [backups.md](backups.md).
- Client IPs, network policies and other hardening: [hardening.md](hardening.md).

## Updating Postgres

The database and backup images are pinned by digest (`postgres:16@sha256:...`), so nothing changes underneath you, and they always match each other. A digest freezes OS packages too, so bump it deliberately, for example monthly or when the dashboard's `k8s-gitops/postgres` entry shows new fixable vulnerabilities:

1. Find the new digest of the multi-arch index: `docker buildx imagetools inspect postgres:16` (the top-level `Digest`), or the Docker Hub tags API.
2. Replace it in both `postgres.yaml` and `backup.yaml`. Stay on major version 16 here: a major upgrade needs a dump and restore.
3. Run a manual backup first: `kubectl create job --from=cronjob/postgres-backup pre-upgrade-backup -n sbom`.
4. Push. Argo CD restarts the pod (a few seconds without the database), and the app reconnects by itself.

## Follow-ups

- Alert on a failed backup Job (with the observability work).
