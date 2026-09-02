# Update: `eegfaktura-eda-xp`

|  |  |
|---|---|
| Source | `~/src/eegfaktura-eda-xp` |
| Image | `localhost:5000/eegfaktura-kep:dev` |
| Deployment | `eegfaktura-eda` |
| Container | `eda` |
| Manifest | [`k8s/61-eda.yaml`](../k8s/61-eda.yaml) |
| First built in | [step 17](../docs/17-billing-eda.md) |
| Deployed in | [step 17](../docs/17-billing-eda.md) |
| Rollout | Recreate — the old pod is stopped before the new one starts, so there is a short outage. That is deliberate: the deployment owns a ReadWriteOnce volume and two pods cannot mount it at once. |

The EDA market-partner service — Scala again, so build-then-retag, and the repo, image and deployment all have different names: `eegfaktura-eda-xp` builds `eegfaktura-kep:dev` into deployment `eegfaktura-eda`. Its configuration is a ConfigMap, not environment variables.

## The whole update, in one paste

```bash
REF=main                       # a branch name, or a release tag
cd ~/src/eegfaktura-eda-xp
git fetch --all --tags
git checkout "$REF"
git pull --ff-only 2>/dev/null || echo "(detached at a tag — nothing to pull)"
git log --oneline -1
```

```bash
cd ~/src/eegfaktura-eda-xp
sbt Docker/publishLocal
IMG=ghcr.io/vfeeg-development/eegfaktura-kep
TAG=$(docker images --format '{{.CreatedAt}}\t{{.Tag}}' "$IMG" | grep -v latest | sort -r | head -1 | cut -f2)
echo "$TAG"; grep -i dockerversion build.sbt
docker tag "$IMG:$TAG" localhost:5000/eegfaktura-kep:dev
docker push localhost:5000/eegfaktura-kep:dev
kubectl rollout restart deployment/eegfaktura-eda
kubectl rollout status  deployment/eegfaktura-eda --timeout=300s
```

If any of that fails, the same thing broken into steps is below.

## 1. Fetch

```bash
cd ~/src/eegfaktura-eda-xp
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
cd ~/src/eegfaktura-eda-xp
sbt Docker/publishLocal
```

sbt names the image for upstream's registry, so retag it into yours. Check the
tag it printed against `dockerVersion` before you overwrite `:dev`:

```bash
IMG=ghcr.io/vfeeg-development/eegfaktura-kep
TAG=$(docker images --format '{{.CreatedAt}}\t{{.Tag}}' "$IMG" | grep -v latest | sort -r | head -1 | cut -f2)
echo "$TAG"; grep -i dockerversion build.sbt
docker tag "$IMG:$TAG" localhost:5000/eegfaktura-kep:dev
```

## 3. Push

```bash
docker push localhost:5000/eegfaktura-kep:dev
```

## 4. Roll out

```bash
kubectl rollout restart deployment/eegfaktura-eda
kubectl rollout status  deployment/eegfaktura-eda --timeout=300s
```

## 5. Verify

```bash
kubectl get pods -l app=eegfaktura-eda
```

```bash
kubectl logs deployment/eegfaktura-eda --tail=40
```

Confirm the running pod is the image you just pushed, not a cached layer:

```bash
kubectl describe pod -l app=eegfaktura-eda | grep -E 'Image:|Image ID:'
```

## Rollback

`kubectl rollout undo` does **not** work here — the previous pod template names
the same mutable `:dev` tag, so it re-pulls the image you are trying to escape.
Tag the build you are replacing *before* you overwrite it:

```bash
SHA=$(git -C ~/src/eegfaktura-eda-xp rev-parse --short HEAD)
docker tag  localhost:5000/eegfaktura-kep:dev localhost:5000/eegfaktura-kep:"$SHA"
docker push localhost:5000/eegfaktura-kep:"$SHA"
```

Then going back is naming it:

```bash
kubectl set image deployment/eegfaktura-eda eda=localhost:5000/eegfaktura-kep:<old-sha>
kubectl rollout status deployment/eegfaktura-eda --timeout=300s
```

## Watch out for

- **`[error]`-prefixed output is not an error.** sbt routes BuildKit's stderr progress through its error channel, so a successful docker build looks like a failure ([step 17](../docs/17-billing-eda.md)).
- **Configuration is in the `eda-config` ConfigMap**, mounted at `/conf` — not in the image and not in env vars. An update that changes a config key needs the ConfigMap updated and the pod restarted; rebuilding the image alone does nothing.
- **The Pekko journal lives on `eda-data`.** Message state survives the restart. If an update changes the journal schema, that is a data migration, not a redeploy.
- **Take the newest tag, not the first** — same trap as admin-backend, above.

---

[← all services](README.md)
