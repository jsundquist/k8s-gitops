# Droplet setup

Notes on how the single-node k3s host was built. Created by hand in the DigitalOcean console, not with Terraform: one droplet doesn't justify the extra tooling or the state-file handling in a public repo.

No IP addresses, keys, usernames or tokens belong in this file.

## Droplet

| Setting | Choice |
|---|---|
| Region | sfo3 |
| Image | Ubuntu LTS |
| Plan | Shared CPU, 2 vCPU, 4 GB RAM |
| Authentication | SSH key only (dedicated ed25519 key) |
| Monitoring | Enabled (free metrics agent) |
| Backups | Off; the repo is the source of truth |
| Startup scripts | None; k3s is installed by hand on purpose |

Powering a droplet off does not stop billing. Only destroying it does.

## Cloud firewall

Applied by tag so future droplets inherit it.

| Direction | Port | Source |
|---|---|---|
| Inbound | 22 (SSH) | anywhere |
| Inbound | 80 (HTTP) | anywhere |
| Inbound | 443 (HTTPS) | anywhere |
| Outbound | all TCP/UDP, ICMP | anywhere |

Everything else is closed, including 6443 (Kubernetes API). The API is reached through an SSH tunnel; `nc -vz <ip> 6443` from outside times out.

SSH is open to the internet, so the host's SSH config is what protects it.

## SSH hardening

`/etc/ssh/sshd_config.d/00-hardening.conf` (the `00-` prefix matters: sshd uses the first value it sees, and cloud-init ships `50-` and `60-` files):

```
PermitRootLogin no
MaxAuthTries 3
AllowUsers <admin user>
PasswordAuthentication no
PubkeyAuthentication yes
```

Process that avoids lockouts:
1. Create the admin user, add it to `sudo`, copy the authorized key over.
2. Log in as that user from a second terminal and run `sudo -v` before changing anything.
3. `sudo sshd -t`, then `sudo sshd -T | grep ...` to confirm the effective values.
4. `sudo systemctl reload ssh` (the unit is `ssh`, not `sshd`).
5. Test a new login, and confirm root is refused. Keep the old session open until both pass.

Automatic security updates are on (`unattended-upgrades`). Automatic reboots are off, so check for `/var/run/reboot-required` after logging in.

## k3s

```
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.37.1+k3s1 INSTALL_K3S_EXEC="--write-kubeconfig-mode 644" sh -s -
```

- Version is pinned so the cluster is reproducible.
- `--write-kubeconfig-mode 644` lets the admin user run `kubectl` without `sudo`. The kubeconfig holds cluster-admin credentials, so this is acceptable only on a single-user host.
- Traefik and the bundled service load balancer stay enabled; the manifests rely on `ingressClassName: traefik`.
- The installer elevates with `sudo` itself. Don't prefix it with `sudo`: `sudo` resets the environment and would silently drop the version pin.
- Clean removal: `/usr/local/bin/k3s-uninstall.sh`.

Baseline after install: one `Ready` node, CoreDNS, Traefik, local-path-provisioner and metrics-server running, about 1.1 GiB used of 3.8 GiB.

## Reaching the cluster from a laptop

The API is not exposed. Copy `/etc/rancher/k3s/k3s.yaml` to a file outside this repo, keep `server: https://127.0.0.1:6443`, and tunnel:

```
ssh -N -L 6443:127.0.0.1:6443 <host alias>
KUBECONFIG=<that file> kubectl get nodes
```

Setting `KUBECONFIG` per command keeps this cluster separate from the local k3d one. Check `kubectl config current-context` before anything destructive: a plain `kubectl` on the laptop points at the local cluster.

## Argo CD on the droplet

Installed into the `argocd` namespace from the upstream install manifest with `kubectl apply --server-side` (the CRDs are too large for client-side apply). The UI is not exposed; reach it with `kubectl port-forward svc/argocd-server -n argocd 8081:443` through the tunnel. The admin password was changed after first login and `argocd-initial-admin-secret` deleted.

The droplet has its own app-of-apps root, separate from the local one:

```
argocd/droplet/root.yaml        # applied by hand once
argocd/droplet/apps/            # cert-manager, cluster-issuers, blog
```

Sync waves order the Applications: cert-manager (0), then cluster-issuers (1), then workloads (2). The issuers Application uses `SkipDryRunOnMissingResource` because the `ClusterIssuer` CRD does not exist until cert-manager has synced.

## DNS and TLS

DNS is hosted at DigitalOcean. Each public hostname has an `A` record pointing at the droplet; no `AAAA` record, since IPv6 is not enabled.

cert-manager is installed as a Helm chart through an Argo CD Application, with the version pinned in `argocd/droplet/apps/cert-manager.yaml`. Two `ClusterIssuer` resources live in `apps/cluster-issuers/`, both using the HTTP-01 solver through Traefik (port 80 must stay open):

- `letsencrypt-staging`: use for any new hostname or change. Browsers do not trust it, but it has generous rate limits.
- `letsencrypt-prod`: switch to it once staging has issued a certificate.

An Ingress opts in with the `cert-manager.io/cluster-issuer` annotation and a `tls` block. To change issuer on an existing hostname, delete the Certificate and its Secret after the annotation change has synced, so a new one is requested.

If a certificate stalls, walk down the chain with `-n <namespace>`: `describe certificate`, then `certificaterequest`, `order`, `challenge`.

## Apps

| Host | Source | Status |
|---|---|---|
| blog.digital-portfolio.net | `apps/blog/` | placeholder nginx until the real image exists |

The apex and `www` hostnames are served by GitHub Pages and are not touched by this cluster.
