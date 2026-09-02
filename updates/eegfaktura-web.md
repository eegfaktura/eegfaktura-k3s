# Update: `eegfaktura-web`

|  |  |
|---|---|
| Source | `~/src/eegfaktura-web` |
| Image | `localhost:5000/vfeeg-web:dev` |
| Deployment | `eegfaktura-web` |
| Container | `web` |
| Manifest | [`k8s/70-web.yaml`](../k8s/70-web.yaml) |
| First built in | [step 18](../docs/18-frontends.md) |
| Deployed in | [step 18](../docs/18-frontends.md) |
| Rollout | RollingUpdate — the new pod becomes ready before the old one is removed, so there is no gap. |

The platform UI. Its Dockerfile packages a **pre-built** `dist/` into a Caddy image and never runs npm, so the build happens on the VM before `docker build` — with pnpm 9 specifically, not whatever corepack offers.

## The whole update, in one paste

```bash
REF=main                       # a branch name, or a release tag
cd ~/src/eegfaktura-web
git fetch --all --tags
git checkout "$REF"
git pull --ff-only 2>/dev/null || echo "(detached at a tag — nothing to pull)"
git log --oneline -1
```

```bash
cd ~/src/eegfaktura-web
rm -rf dist
corepack pnpm@9.12.1 install
corepack pnpm@9.12.1 run build
ls dist
docker build -t localhost:5000/vfeeg-web:dev ~/src/eegfaktura-web
docker push localhost:5000/vfeeg-web:dev
kubectl rollout restart deployment/eegfaktura-web
kubectl rollout status  deployment/eegfaktura-web --timeout=300s
```

If any of that fails, the same thing broken into steps is below.

## 1. Fetch

```bash
cd ~/src/eegfaktura-web
git status --short
```

Anything listed is dealt with before pulling — see
[the summary](README.md#1-fetch) for the two repos that dirty themselves.

```bash
git fetch --all --tags
git checkout "$REF"          # branch or tag
git pull --ff-only           # branches only; a tag is already fixed
git log --oneline -1
```

## 2. Build

First, on the VM:

```bash
cd ~/src/eegfaktura-web
rm -rf dist
corepack pnpm@9.12.1 install
corepack pnpm@9.12.1 run build
ls dist
```

Then the image:

```bash
docker build -t localhost:5000/vfeeg-web:dev ~/src/eegfaktura-web
```

## 3. Push

```bash
docker push localhost:5000/vfeeg-web:dev
```

## 4. Roll out

```bash
kubectl rollout restart deployment/eegfaktura-web
kubectl rollout status  deployment/eegfaktura-web --timeout=300s
```

## 5. Verify

```bash
kubectl get pods -l app=eegfaktura-web
```

```bash
kubectl exec deployment/eegfaktura-web -- ls /var/www/html/vfeeg-web/
```

Confirm the running pod is the image you just pushed, not a cached layer:

```bash
kubectl describe pod -l app=eegfaktura-web | grep -E 'Image:|Image ID:'
```

## Rollback

`kubectl rollout undo` does **not** work here — the previous pod template names
the same mutable `:dev` tag, so it re-pulls the image you are trying to escape.
Tag the build you are replacing *before* you overwrite it:

```bash
SHA=$(git -C ~/src/eegfaktura-web rev-parse --short HEAD)
docker tag  localhost:5000/vfeeg-web:dev localhost:5000/vfeeg-web:"$SHA"
docker push localhost:5000/vfeeg-web:"$SHA"
```

Then going back is naming it:

```bash
kubectl set image deployment/eegfaktura-web web=localhost:5000/vfeeg-web:<old-sha>
kubectl rollout status deployment/eegfaktura-web --timeout=300s
```

## Watch out for

- **`rm -rf dist` is the important line.** `dist/` is git-ignored and survives `git switch`. Skip the removal, let the new branch fail to build, and `docker build` will package the *previous* frontend and push it as `:dev` — a green rollout serving your old code.
- **pnpm 9, not 10 or 11.** The repo declares no `packageManager`, so corepack would install the newest, which ignores the `pnpm.overrides` pin on `form-data` and blocks the postinstall `esbuild` needs ([known problems #16](../docs/known-problems.md#16-eegfaktura-web-does-not-pin-its-package-manager)). `corepack pnpm@9.12.1 …` pins it per invocation.
- **A newer pnpm leaves debris.** If one ever ran here you will have an untracked `pnpm-workspace.yaml` and a rewritten `pnpm-lock.yaml`; remove the first, `git checkout` the second.
- **Hard-refresh before believing anything.** The browser caches the bundle, so a successful rollout can look like nothing happened.

---

[← all services](README.md)
