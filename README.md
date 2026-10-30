# k8s-gitops

A hands-on project for learning Kubernetes in practice. The goal is to run my Astro blog and my SBOM dashboard on a single-node [k3s](https://k3s.io) cluster on a DigitalOcean droplet, managed with [Argo CD](https://argo-cd.readthedocs.io) and [Helm](https://helm.sh).

This repo is public, so no plaintext secrets are ever committed. Argo CD pulls from it over HTTPS with no deploy key.

> Work in progress: I'm learning as I go, and the history shows it.

## Roadmap

- [ ] 1. Reset: README stating the goal
- [ ] 2. Practice locally with k3d: nginx Deployment, Service and Ingress by hand
- [ ] 3. Install Argo CD locally and sync the nginx app from this repo
- [ ] 4. Structure the repo for multiple apps (app-of-apps or ApplicationSet)
- [ ] 5. Create the droplet, install k3s, lock it down (firewall, SSH keys)
- [ ] 6. Argo CD on the droplet, then ingress and TLS (Traefik + cert-manager)
- [ ] 7. Ship the blog: Dockerfile, GitHub Actions to GHCR, manifests or Helm chart
- [ ] 8. Ship the SBOM dashboard the same way
- [ ] 9. Hardening and observability
