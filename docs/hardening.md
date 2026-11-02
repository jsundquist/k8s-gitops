# Hardening

Stage 9 notes: what has been done, why, and what is left.

## Done

| Item | Where |
|---|---|
| Nightly encrypted Postgres backups to DigitalOcean Spaces | [backups.md](backups.md) |
| Real client IPs reach the apps | below |
| Default-deny network policies for the `blog` and `sbom` namespaces | below |
| Non-root pods, read-only root filesystems, dropped capabilities, `RuntimeDefault` seccomp | `apps/blog`, `apps/sbom` |
| Pre-commit gitleaks, with a narrow allowlist for Sealed Secrets ciphertext | `lefthook.yml`, `.gitleaks.toml` |
| CI: gitleaks history scan, manifest validation, actionlint, zizmor; Dependabot with a 7-day cooldown | `.github/workflows/ci.yml`, `.github/dependabot.yml` |
| Weekly SBOM upload for every deployed image, watched by the SBOM dashboard | `.github/workflows/cluster-sbom.yml` |
| Postgres and rclone pinned by digest; CI fails any other unpinned image except the three the release workflows re-pin | `apps/sbom/postgres.yaml`, `apps/sbom/backup.yaml`, `.github/workflows/ci.yml` |
| `SECURITY.md`, private vulnerability reporting, actions required to be SHA-pinned | `SECURITY.md`, repository settings |

## Client IPs

The sbom app rate-limits per client IP, taken from the leftmost `X-Forwarded-For` value when `TRUST_PROXY=true`. That is safe only if Traefik overwrites an inbound header and the real address reaches Traefik. A throwaway `whoami` pod showed:

- Traefik **does** overwrite a forged `X-Forwarded-For`.
- With the default `externalTrafficPolicy: Cluster`, k3s's load balancer rewrote the source address, so every client arrived as the node's bridge IP (`10.42.0.1`) and shared one rate-limit bucket (including the sign-in one).

The fix is `externalTrafficPolicy: Local` on the Traefik service, set declaratively with a `HelmChartConfig` named `traefik` in `kube-system` (`apps/traefik-config/`), because k3s installs Traefik with its own Helm controller. After the change the same test showed the real address, and a forged header was still replaced. Traefik was not restarted and there was no downtime.

## Network policies

k3s enforces NetworkPolicies out of the box. Each app namespace has a `default-deny` policy for ingress and egress, and explicit allows on top:

| Traffic | Allowed by |
|---|---|
| Traefik to blog (8080) and sbom web (3000) | `allow-traefik-to-*` |
| Traefik to cert-manager's temporary HTTP-01 solver pods (8089), needed for renewals | `allow-traefik-to-acme-solver` |
| Any sbom pod to CoreDNS | `allow-dns` |
| web, worker, migrate, backup to Postgres (5432), and Postgres only accepts those | `postgres-ingress`, `allow-egress-to-postgres` |
| web, worker, backup to the public internet on 443 only | `allow-egress-https-external` |
| backup to the Spaces endpoint inside the VPC | `allow-backup-to-spaces-vpc-endpoint` |

Verified after rollout: allowed paths work (GitHub, OSV, Postgres, a full backup), and these are blocked: web to internet port 80, web to the cloud metadata address, web to another namespace, and a pod in another namespace to Postgres or the blog.

Lessons:

- **NetworkPolicy cannot match hostnames.** "Public HTTPS" is `0.0.0.0/0` on 443 minus the private ranges and the metadata address (`169.254.0.0/16`).
- **Spaces resolves to a private address from inside the droplet.** DigitalOcean's resolver answers `sfo3.digitaloceanspaces.com` with an address in the VPC range (`10.124.0.0/20`), while the public internet gets a public IP. The "public 443" rule excludes private ranges, so the backup job needs its own narrow rule for the VPC range. Confirm the range on the console's VPC page if the droplet is ever recreated.
- **A new pod's rules take about a second to apply.** Connections made in the first moments of a pod's life are refused. Anything that connects immediately (the backup's dump step) must retry. Web and worker retry on their own, and Jobs have `backoffLimit`.
- **`ECONNREFUSED`, not a timeout.** k3s's policy engine rejects instead of dropping, so blocked connections fail fast.

Not covered yet: `argocd` (the upstream install ships its own ingress policies), `cert-manager`, and `kube-system`. A default-deny there needs careful allows for the API server, webhooks and ACME self-checks, so it is deferred.

## Left to do

- Swap file and resource headroom (4 GB, no swap).
- Observability: metrics, logs and alerting (including failed backups).
