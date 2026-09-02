# Update: `eegfaktura-admin-backend`

|  |  |
|---|---|
| Source | `~/src/eegfaktura-admin-backend` |
| Image | `localhost:5000/eeg-registration-backend:dev` |
| Deployment | `eegfaktura-admin-backend` |
| Container | `admin-backend` |
| Manifest | [`k8s/50-admin-backend.yaml`](../k8s/50-admin-backend.yaml) |
| First built in | [step 16](../docs/16-admin-backend.md) |
| Deployed in | [step 16](../docs/16-admin-backend.md) |
| Rollout | RollingUpdate — the new pod becomes ready before the old one is removed, so there is no gap. |

Scala. sbt builds the image itself and names it for **upstream's** registry, so every update needs a retag before the push. The tag comes from `dockerVersion` in `build.sbt` and is exactly the thing a new release changes.

## The whole update, in one paste

```bash
REF=main                       # a branch name, or a release tag
cd ~/src/eegfaktura-admin-backend
git fetch --all --tags
git checkout "$REF"
git pull --ff-only 2>/dev/null || echo "(detached at a tag — nothing to pull)"
git log --oneline -1
```

```bash
cd ~/src/eegfaktura-admin-backend
sbt Docker/publishLocal
IMG=ghcr.io/vfeeg-development/eeg-registration-backend
TAG=$(docker images --format '{{.CreatedAt}}\t{{.Tag}}' "$IMG" | grep -v latest | sort -r | head -1 | cut -f2)
echo "$TAG"; grep -i dockerversion build.sbt
docker tag "$IMG:$TAG" localhost:5000/eeg-registration-backend:dev
docker push localhost:5000/eeg-registration-backend:dev
kubectl rollout restart deployment/eegfaktura-admin-backend
kubectl rollout status  deployment/eegfaktura-admin-backend --timeout=300s
```

If any of that fails, the same thing broken into steps is below.

## 1. Fetch

```bash
cd ~/src/eegfaktura-admin-backend
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
cd ~/src/eegfaktura-admin-backend
sbt Docker/publishLocal
```

sbt names the image for upstream's registry, so retag it into yours. Check the
tag it printed against `dockerVersion` before you overwrite `:dev`:

```bash
IMG=ghcr.io/vfeeg-development/eeg-registration-backend
TAG=$(docker images --format '{{.CreatedAt}}\t{{.Tag}}' "$IMG" | grep -v latest | sort -r | head -1 | cut -f2)
echo "$TAG"; grep -i dockerversion build.sbt
docker tag "$IMG:$TAG" localhost:5000/eeg-registration-backend:dev
```

## 3. Push

```bash
docker push localhost:5000/eeg-registration-backend:dev
```

## 4. Roll out

```bash
kubectl rollout restart deployment/eegfaktura-admin-backend
kubectl rollout status  deployment/eegfaktura-admin-backend --timeout=300s
```

## 5. Verify

```bash
kubectl get pods -l app=eegfaktura-admin-backend
```

```bash
kubectl logs deployment/eegfaktura-admin-backend --tail=40
```

Confirm the running pod is the image you just pushed, not a cached layer:

```bash
kubectl describe pod -l app=eegfaktura-admin-backend | grep -E 'Image:|Image ID:'
```

## Rollback

`kubectl rollout undo` does **not** work here — the previous pod template names
the same mutable `:dev` tag, so it re-pulls the image you are trying to escape.
Tag the build you are replacing *before* you overwrite it:

```bash
SHA=$(git -C ~/src/eegfaktura-admin-backend rev-parse --short HEAD)
docker tag  localhost:5000/eeg-registration-backend:dev localhost:5000/eeg-registration-backend:"$SHA"
docker push localhost:5000/eeg-registration-backend:"$SHA"
```

Then going back is naming it:

```bash
kubectl set image deployment/eegfaktura-admin-backend admin-backend=localhost:5000/eeg-registration-backend:<old-sha>
kubectl rollout status deployment/eegfaktura-admin-backend --timeout=300s
```

## Watch out for

- **Take the newest tag, not the first.** After an update both builds sit in `docker images`, and [step 16](../docs/16-admin-backend.md)'s `head -1` can hand you the previous release. Pushing that under `:dev` is a silent no-op update. The `sort -r` above is why the two commands differ.
- **It needs `KEYCLOAK_ADMIN_CLI_SECRET`**, from the `eegfaktura-admin-cli` secret created in [step 12.2](../docs/12-realm-config.md#122-rotate-the-admin-cli-secret). The secret is not rotated by an update — but if registration starts returning 401 afterwards, that secret is the first thing to check.
- **sbt may run out of heap** on a 16 GB VM. `SBT_OPTS=-Xmx4G sbt Docker/publishLocal` is the fix from step 16.

---

[← all services](README.md)
