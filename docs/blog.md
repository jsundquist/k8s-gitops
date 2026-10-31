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
- It expires. When it does, the daily `bump` job fails and the site stops updating (it keeps serving the last version). Set a calendar reminder before the expiry date.

## Follow-ups

- `git pull --rebase` before the push in the `bump` job, in case two runs overlap.
- Retention for old GHCR images. One is added every day and only the latest is used.
