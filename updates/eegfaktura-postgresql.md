# Update: `eegfaktura-postgresql`

|  |  |
|---|---|
| Source | `~/src/eegfaktura-postgresql` |
| Image | `localhost:5000/eegfaktura-postgresql:dev` |
| Deployment | `eegfaktura-postgresql` |
| Container | `postgres` |
| Manifest | [`k8s/10-postgres.yaml`](../k8s/10-postgres.yaml) |
| First built in | [step 08](../docs/08-infra-images.md) |
| Deployed in | [step 10](../docs/10-postgres.md) |
| Rollout | Recreate — the old pod is stopped before the new one starts, so there is a short outage. That is deliberate: the deployment owns a ReadWriteOnce volume and two pods cannot mount it at once. |

The database everything else depends on. A rebuild here is almost never what you actually want: the image's value is in its init scripts, and those run **once**, against an empty data directory. Read [Watch out for](#watch-out-for) before you push anything.

## The whole update, in one paste

```bash
REF=main                       # a branch name, or a release tag
cd ~/src/eegfaktura-postgresql
git fetch --all --tags
git checkout "$REF"
git pull --ff-only 2>/dev/null || echo "(detached at a tag — nothing to pull)"
git log --oneline -1
```

```bash
docker build -t localhost:5000/eegfaktura-postgresql:dev ~/src/eegfaktura-postgresql
docker push localhost:5000/eegfaktura-postgresql:dev
kubectl rollout restart deployment/eegfaktura-postgresql
kubectl rollout status  deployment/eegfaktura-postgresql --timeout=300s
```

If any of that fails, the same thing broken into steps is below.

## 1. Fetch

```bash
cd ~/src/eegfaktura-postgresql
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
docker build -t localhost:5000/eegfaktura-postgresql:dev ~/src/eegfaktura-postgresql
```

## 3. Push

```bash
docker push localhost:5000/eegfaktura-postgresql:dev
```

## 4. Roll out

```bash
kubectl rollout restart deployment/eegfaktura-postgresql
kubectl rollout status  deployment/eegfaktura-postgresql --timeout=300s
```

## 5. Verify

```bash
kubectl get pods -l app=eegfaktura-postgresql
```

```bash
kubectl exec deployment/eegfaktura-postgresql -- psql -U postgres -l
```

Confirm the running pod is the image you just pushed, not a cached layer:

```bash
kubectl describe pod -l app=eegfaktura-postgresql | grep -E 'Image:|Image ID:'
```

## Rollback

`kubectl rollout undo` does **not** work here — the previous pod template names
the same mutable `:dev` tag, so it re-pulls the image you are trying to escape.
Tag the build you are replacing *before* you overwrite it:

```bash
SHA=$(git -C ~/src/eegfaktura-postgresql rev-parse --short HEAD)
docker tag  localhost:5000/eegfaktura-postgresql:dev localhost:5000/eegfaktura-postgresql:"$SHA"
docker push localhost:5000/eegfaktura-postgresql:"$SHA"
```

Then going back is naming it:

```bash
kubectl set image deployment/eegfaktura-postgresql postgres=localhost:5000/eegfaktura-postgresql:<old-sha>
kubectl rollout status deployment/eegfaktura-postgresql --timeout=300s
```

## Watch out for

- **A new image will not change an existing database.** Everything in `docker-entrypoint-initdb.d/` — users, databases, the seeded `base` and `eda` schemas, the `uuid-ossp` extension — runs only when `PGDATA` is empty. The PVC is not. The rebuilt image starts cleanly, logs nothing unusual, and serves the schema it already had.
- **So apply schema changes yourself**, with `psql`, or delete the PVC and lose the data. There is no third option.
- **If the update seeds something new**, re-read [step 10.4](../docs/10-postgres.md#104-hand-the-seeded-schemas-to-their-owners) — objects the image creates are owned by `postgres`, and the services' own migrations cannot reconcile with tables they do not own.
- **Everything else fails while this is down.** Restart it first and on its own, then watch the dependants recover: `kubectl get pods -w`.

---

[← all services](README.md)
