# k8s-gitops

A hands-on project for learning Kubernetes in practice. The goal is to run my Astro blog and my SBOM dashboard on a single-node [k3s](https://k3s.io) cluster on a DigitalOcean droplet, managed with [Argo CD](https://argo-cd.readthedocs.io) and [Helm](https://helm.sh).

This repo is public, so no plaintext secrets are ever committed. Argo CD pulls from it over HTTPS with no deploy key.

> Work in progress: I'm learning as I go, and the history shows it.

## Repo layout

```
argocd/
  root.yaml     # app-of-apps root; the only manifest applied by hand
  apps/         # one Argo CD Application per app, watched by the root
apps/
  nginx/        # plain manifests: Deployment, Service, Ingress
  whoami/       # throwaway second app used to test multi-app structure
  blog/, sbom/  # the real apps on the droplet (Deployments, Postgres, network policies, sealed secrets)
  traefik-config/ # HelmChartConfig that keeps real client IPs
argocd/droplet/ # same pattern for the DigitalOcean cluster: root, cert-manager, sealed-secrets, issuers, traefik-config, blog, sbom
docs/           # droplet, blog, SBOM dashboard, backups and hardening notes
lefthook.yml    # pre-commit hook: gitleaks scans staged changes
```

Bootstrap on a cluster that already has Argo CD: `kubectl apply -f argocd/root.yaml`. Argo CD creates everything else from git.

## Tools and CI

Local pre-commit hook (`lefthook.yml`) and CI run the same secret scan, so versions are kept in step:

| Tool | Version | Where |
|---|---|---|
| gitleaks | 8.30.1 | `brew install gitleaks`; pinned by version and sha256 in `.github/workflows/ci.yml` |
| lefthook | 2.2.2 | `brew install lefthook`, then `lefthook install` |

`.github/workflows/ci.yml` runs on every push and pull request: a gitleaks scan of the whole history, a kubeconform validation of every manifest (Kustomize apps are built first), and a check that every Argo CD Application points at a path that exists. Dependabot keeps the pinned GitHub Actions current. The `.gitleaks.toml` allowlist covers only Sealed Secrets ciphertext. GitHub's own secret scanning and push protection are also enabled on the repository.

## Roadmap

- [x] 1. Reset: README stating the goal
- [x] 2. Practice locally with k3d: nginx Deployment, Service and Ingress by hand
- [x] 3. Install Argo CD locally and sync the nginx app from this repo
- [x] 4. Structure the repo for multiple apps (app-of-apps or ApplicationSet)
- [x] 5. Create the droplet, install k3s, lock it down (firewall, SSH keys) ([notes](docs/droplet.md))
- [x] 6. Argo CD on the droplet, then ingress and TLS (Traefik + cert-manager)
- [x] 7. Ship the blog: Dockerfile, GitHub Actions to GHCR, manifests or Helm chart ([notes](docs/blog.md))
- [x] 8. Ship the SBOM dashboard the same way ([notes](docs/sbom.md))
- [ ] 9. Hardening and observability ([notes](docs/hardening.md), [backups](docs/backups.md))
