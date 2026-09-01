# Step 20 — Recreate the build pipeline

The cluster works by hand. This step automates it — the part that turns "it
runs" into understanding how it ships.

## 20.1 What upstream does

From `eegfaktura-backend/.github/workflows/rolling-release.yml`, three stages:

1. **Build** — `docker/build-push-action` → `ghcr.io/vfeeg-development/<image>`,
   tagged via `docker/metadata-action`: `latest` on the default branch,
   `sha-<short>`, semver on tags, branch/PR refs. Plus
   `actions/attest-build-provenance`.
2. **Dispatch** — `repository_dispatch` to the private platform repo,
   `event_type: deploy-backend`, payload carrying `image_tag` and `source_sha`.
3. **Deploy** — the platform repo applies `k8s/30-backend.yaml`.

Two properties worth copying: PRs **build but never push**, so forks cannot
publish images (login only happens on `push` events); and `preview/**` branches
deploy pinned to their `sha-` tag rather than `latest`, so a preview never
disturbs `main`.

That the workflow references plain numbered manifests, not Helm, is why this
repo's `k8s/` directory is shaped the way it is.

## 20.2 The problem with copying it directly

> [!WARNING]
> **GitHub-hosted runners cannot reach your VM.** It is behind NAT on a private
> network, and exposing the Kubernetes API to the internet to fix that is a poor
> trade.

| Option | How | Trade-off |
|---|---|---|
| **Self-hosted runner** on the VM | The runner has local cluster and registry access; the job ends with `kubectl apply` | Simple, mirrors upstream's push model — but the runner executes repository code on your VM |
| **Pull-based GitOps** (Flux or Argo CD in-cluster) | CI only builds and pushes; the cluster watches a manifest repo and reconciles | No inbound access needed; the better durable answer |

For learning the *upstream* model, the self-hosted runner is the closer
analogue. For a setup you keep, pull-based is better.

## 20.3 Stage 1 — build and push

Start here regardless of which deploy model you pick. In your fork of a service
repo, `.github/workflows/build.yml`:

```yaml
name: build
on:
  push:
    branches: [main, master, 'preview/**']
    tags: ['v*']
  pull_request:

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository_owner }}/vfeeg-backend

jobs:
  build:
    runs-on: ubuntu-latest
    permissions: {contents: read, packages: write}
    steps:
      - uses: actions/checkout@v4
      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=raw,value=latest,enable={{is_default_branch}}
            type=sha,format=short,prefix=sha-
      - if: github.event_name == 'push'
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: ${{ github.event_name == 'push' }}
          tags: ${{ steps.meta.outputs.tags }}
```

> [!NOTE]
> **`GITHUB_TOKEN` suffices here.** Upstream needs a PAT because it pushes
> *cross-org*. Pushing into your own namespace needs only the built-in token
> with `packages: write`.

> [!IMPORTANT]
> For `eegfaktura-energystore` and the two frontends, this workflow alone will
> not produce a working image — their Dockerfiles need a preceding codegen or
> npm step (steps [15](15-backend-energystore.md) and
> [18](18-frontends.md)). Add that step to the job before
> `docker/build-push-action`, exactly as you ran it by hand.

## 20.4 Pulling private images into k3s

If your package is private, k3s needs credentials:

```bash
kubectl create secret docker-registry ghcr \
  --docker-server=ghcr.io \
  --docker-username=<your-github-user> \
  --docker-password=<PAT with read:packages>
```

```bash
kubectl patch serviceaccount default -p '{"imagePullSecrets":[{"name":"ghcr"}]}'
```

Then switch a Deployment's image from `localhost:5000/…:dev` to
`ghcr.io/<you>/vfeeg-backend:latest` and confirm the pull works.

## 20.5 Stage 3 — deploy

With a self-hosted runner on the VM, append to the same workflow:

```yaml
  deploy:
    needs: build
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
      - run: |
          kubectl set image deployment/eegfaktura-backend \
            backend=ghcr.io/${{ github.repository_owner }}/vfeeg-backend:sha-${GITHUB_SHA::7} \
            -n eegfaktura
          kubectl rollout status deployment/eegfaktura-backend -n eegfaktura --timeout=300s
```

> [!TIP]
> **Deploy the `sha-` tag, not `latest`.** `kubectl set image` with an immutable
> tag makes rollouts explicit and rollback trivial (`kubectl rollout undo`).
> With `latest` the tag does not change, so Kubernetes may not restart anything
> at all — a classic silent no-op deploy.

## 20.6 Where to go next

- Repeat for a second service, to feel the multi-repo shape
- Add dependency scanning — `govulncheck` for the Go services, remembering that
  it needs codegen first or it cannot build the package graph
- Try the preview pattern: `preview/**` → pinned deploy into a second namespace
- If this becomes permanent, move to Flux or Argo CD so the cluster pulls rather
  than being pushed to, and drop the self-hosted runner

## Done when

A push to `main` builds an image, pushes it to ghcr.io, and the cluster ends up
running that exact SHA — with no manual step.

---

← back to the [README](../README.md) · the defects behind all of this are in
[known-problems.md](known-problems.md)
