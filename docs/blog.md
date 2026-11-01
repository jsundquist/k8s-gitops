# Blog deployment

The Astro blog is served from the droplet at `blog.digital-portfolio.net`. The same content is also published to GitHub Pages at the apex and `www`; the two are independent.

## Pipeline

```
push to main (or daily cron)
  -> build job: pnpm build, upload dist/ as an artifact
  -> deploy job: GitHub Pages
  -> image job: Dockerfile packages dist/ into nginx-unprivileged, pushed to GHCR
  -> bump job: rewrites images[0].newTag in apps/blog/kustomization.yaml in this repo
  -> Argo CD syncs apps/blog -> rolling update on the droplet
```

The source, `Dockerfile`, `nginx.conf` and workflow live in the blog repository (`jsundquist.github.io`). This repository only holds the manifests.

## Decisions

- **One build, two outputs.** The image packages the same `dist/` that Pages publishes, so the two sites are byte-identical.
- **Daily rebuild.** The site embeds git activity, so it is rebuilt every day and the image tag changes with it. Tags are `<short sha>-<run number>`, which stays unique when the same commit is rebuilt. The cost is one bot commit per day here (`chore(blog): deploy <tag>`), which can be filtered out of `git log`.
- **Image tag lives in `kustomization.yaml`, not `deployment.yaml`.** The Deployment references the image name only; Kustomize injects the tag. The workflow edits one field and never touches the manifest.
- **Public image.** The image contains only the built static site, which is public anyway, so the cluster needs no image pull secret. A private package would require storing a token in the cluster, which in a public repo means a secrets tool first. Revisit for any app that ships non-public content.
- **Non-root container.** `nginx-unprivileged` listens on 8080, so `containerPort` is 8080 and the Service uses the named port `http`.
- **Probes.** Readiness and liveness probes hit `/`. The readiness probe is what keeps the daily rolling update from dropping traffic.

## The deploy token

The `bump` job pushes to this repository with a fine-grained personal access token stored as the `GITOPS_TOKEN` Actions secret in the blog repo.

- Scope: this repository only, `Contents: Read and write`, nothing else.
- This token can change what runs on the cluster, because Argo CD deploys whatever lands on `main`. Keep the scope narrow.
- It expires. When it does, the daily `bump` job fails and the site stops updating (it keeps serving the last version). A separate `token-expiry` job in the workflow reads the token's expiry from the API on every run and fails (so GitHub emails a failed-workflow notice) when fewer than 14 days remain, or when the token is rejected. Keep a calendar reminder anyway.

## Resilience and cleanup

- **Bump races.** The `bump` job runs `git pull --rebase` before pushing and retries up to 3 times with a backoff, because this repository also receives human commits. It exits cleanly if the tag is already pinned.
- **Image retention.** The `prune` job runs after `bump` and keeps the newest 10 tagged versions of the GHCR package. It never deletes the tag it just built or the tag currently pinned in this repository (covers manual rollbacks), and it only deletes untagged manifests that are more than an hour older than the cutoff. It needs the package to grant the blog repository Admin under "Manage Actions access".
- **Prune is safe by default.** Scheduled and push runs only delete when the repository variable `GHCR_PRUNE_ENABLED` is `true`; otherwise they log what they would delete. To enable it: run the workflow manually (the `dry_run` input defaults to true), read the "to delete" list once there are more than 10 tagged versions, and only then set the variable.
