# Update: `eegfaktura-backend`

|  |  |
|---|---|
| Source | `~/src/eegfaktura-backend` |
| Image | `localhost:5000/vfeeg-backend:dev` |
| Deployment | `eegfaktura-backend` |
| Container | `backend` |
| Manifest | [`k8s/40-backend.yaml`](../k8s/40-backend.yaml) |
| First built in | [step 15](../docs/15-backend-energystore.md) |
| Deployed in | [step 15](../docs/15-backend-energystore.md) |
| Rollout | Recreate — the old pod is stopped before the new one starts, so there is a short outage. That is deliberate: the deployment owns a ReadWriteOnce volume and two pods cannot mount it at once. |

The core Go service. Its Dockerfile installs the Go toolchain, protoc and three plugins and runs codegen before compiling, so unlike energystore it builds from a clean clone with no pre-step. It also runs its own database migrations at startup, which makes the rollout the moment schema changes land.

## The whole update, in one paste

```bash
REF=main                       # a branch name, or a release tag
cd ~/src/eegfaktura-backend
git fetch --all --tags
git checkout "$REF"
git pull --ff-only 2>/dev/null || echo "(detached at a tag — nothing to pull)"
git log --oneline -1
```

```bash
docker build -t localhost:5000/vfeeg-backend:dev ~/src/eegfaktura-backend
docker push localhost:5000/vfeeg-backend:dev
kubectl rollout restart deployment/eegfaktura-backend
kubectl rollout status  deployment/eegfaktura-backend --timeout=300s
```

If any of that fails, the same thing broken into steps is below.

## 1. Fetch

```bash
cd ~/src/eegfaktura-backend
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
docker build -t localhost:5000/vfeeg-backend:dev ~/src/eegfaktura-backend
```

## 3. Push

```bash
docker push localhost:5000/vfeeg-backend:dev
```

## 4. Roll out

```bash
kubectl rollout restart deployment/eegfaktura-backend
kubectl rollout status  deployment/eegfaktura-backend --timeout=300s
```

## 5. Verify

```bash
kubectl get pods -l app=eegfaktura-backend
```

```bash
kubectl logs deployment/eegfaktura-backend --tail=50 | grep -iE 'migrat|listen|error'
```

Confirm the running pod is the image you just pushed, not a cached layer:

```bash
kubectl describe pod -l app=eegfaktura-backend | grep -E 'Image:|Image ID:'
```

## Rollback

`kubectl rollout undo` does **not** work here — the previous pod template names
the same mutable `:dev` tag, so it re-pulls the image you are trying to escape.
Tag the build you are replacing *before* you overwrite it:

```bash
SHA=$(git -C ~/src/eegfaktura-backend rev-parse --short HEAD)
docker tag  localhost:5000/vfeeg-backend:dev localhost:5000/vfeeg-backend:"$SHA"
docker push localhost:5000/vfeeg-backend:"$SHA"
```

Then going back is naming it:

```bash
kubectl set image deployment/eegfaktura-backend backend=localhost:5000/vfeeg-backend:<old-sha>
kubectl rollout status deployment/eegfaktura-backend --timeout=300s
```

## Watch out for

- **Migrations run during the rollout.** A new migration in the update applies as the pod starts. Watch the logs rather than assuming: `kubectl logs deployment/eegfaktura-backend -f`.
- **A failed migration leaves `golang-migrate` dirty**, and every later start refuses to continue with `Dirty database version N`. Recovery is in [step 15](../docs/15-backend-energystore.md) — you must fix the cause and clear the flag; restarting alone will not do it.
- **Schema ownership is the usual cause.** If a migration fails on `CREATE INDEX` or `must be owner of table`, it is [step 10.4](../docs/10-postgres.md#104-hand-the-seeded-schemas-to-their-owners), not your code.
- **It serves gRPC on 9092 to energystore** as well as REST through the ingress. If energystore starts erroring after a backend update, check that the proto contract still matches — both sides generate from `.proto` files that live in separate repos.

---

[← all services](README.md)
