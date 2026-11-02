# Security Policy

## Reporting a vulnerability

Please report vulnerabilities privately through GitHub's private vulnerability reporting:
[Report a vulnerability](https://github.com/jsundquist/k8s-gitops/security/advisories/new).

Do not open a public issue or pull request for a suspected vulnerability. This repository holds
Kubernetes manifests only. Sealed Secrets in it are encrypted, but if you believe a plaintext
secret has been committed, report it the same way.

## Supported versions

Only the latest commit on `main` is supported. There are no tagged releases.

## Response time

Reports are acknowledged within 7 days. A fix or mitigation is targeted within 30 days, depending
on severity. This is a personal project maintained on a best-effort basis.
