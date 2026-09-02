# Update: `eegfaktura-filestore`

|  |  |
|---|---|
| Source | `~/src/eegfaktura-filestore` |
| Image | `localhost:5000/eegfaktura-filestore:dev` |
| Deployment | `eegfaktura-filestore` |
| Container | `filestore` |
| Manifest | [`k8s/32-filestore.yaml`](../k8s/32-filestore.yaml) |
| First built in | [step 14](../docs/14-support-services.md) |
| Deployed in | [step 14](../docs/14-support-services.md) |
| Rollout | Recreate — the old pod is stopped before the new one starts, so there is a short outage. That is deliberate: the deployment owns a ReadWriteOnce volume and two pods cannot mount it at once. |

Python/FastAPI file storage, built straight from its Dockerfile. It holds uploaded workbooks and generated documents on a volume, and reads its database password from a mounted secret.

## The whole update, in one paste

```bash
REF=main                       # a branch name, or a release tag
cd ~/src/eegfaktura-filestore
git fetch --all --tags
git checkout "$REF"
git pull --ff-only 2>/dev/null || echo "(detached at a tag — nothing to pull)"
git log --oneline -1
```

```bash
docker build -t localhost:5000/eegfaktura-filestore:dev ~/src/eegfaktura-filestore
docker push localhost:5000/eegfaktura-filestore:dev
kubectl rollout restart deployment/eegfaktura-filestore
kubectl rollout status  deployment/eegfaktura-filestore --timeout=300s
```

If any of that fails, the same thing broken into steps is below.

## 1. Fetch

```bash
cd ~/src/eegfaktura-filestore
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

Nothing to do first — the Dockerfile is self-contained.

```bash
docker build -t localhost:5000/eegfaktura-filestore:dev ~/src/eegfaktura-filestore
```

## 3. Push

```bash
docker push localhost:5000/eegfaktura-filestore:dev
```

## 4. Roll out

```bash
kubectl rollout restart deployment/eegfaktura-filestore
kubectl rollout status  deployment/eegfaktura-filestore --timeout=300s
```

## 5. Verify

```bash
kubectl get pods -l app=eegfaktura-filestore
```

```bash
kubectl logs deployment/eegfaktura-filestore --tail=30
```

Confirm the running pod is the image you just pushed, not a cached layer:

```bash
kubectl describe pod -l app=eegfaktura-filestore | grep -E 'Image:|Image ID:'
```

## Rollback

`kubectl rollout undo` does **not** work here — the previous pod template names
the same mutable `:dev` tag, so it re-pulls the image you are trying to escape.
Tag the build you are replacing *before* you overwrite it:

```bash
SHA=$(git -C ~/src/eegfaktura-filestore rev-parse --short HEAD)
docker tag  localhost:5000/eegfaktura-filestore:dev localhost:5000/eegfaktura-filestore:"$SHA"
docker push localhost:5000/eegfaktura-filestore:"$SHA"
```

Then going back is naming it:

```bash
kubectl set image deployment/eegfaktura-filestore filestore=localhost:5000/eegfaktura-filestore:<old-sha>
kubectl rollout status deployment/eegfaktura-filestore --timeout=300s
```

## Watch out for

- **The volume survives; the pod does not.** Uploads already on `filestore-data` are safe. An upload in flight during the restart is not.
- **It is the only service whose ingress keeps its path prefix.** If an update changes its routes, the StripPrefix reasoning in [step 13](../docs/13-ingress.md) is what to re-read — do not add a middleware to it by reflex.
- **`FILESTORE_CREATE_UNKNOWN_*` treats any non-empty string as true.** Disabling one means setting it to `""`, not `"false"`.

---

[← all services](README.md)
