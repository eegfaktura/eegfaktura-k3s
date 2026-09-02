# Update: `eegfaktura-billing`

|  |  |
|---|---|
| Source | `~/src/eegfaktura-billing` |
| Image | `localhost:5000/eegfaktura-billing:dev` |
| Deployment | `eegfaktura-billing` |
| Container | `billing` |
| Manifest | [`k8s/60-billing.yaml`](../k8s/60-billing.yaml) |
| First built in | [step 17](../docs/17-billing-eda.md) |
| Deployed in | [step 17](../docs/17-billing-eda.md) |
| Rollout | RollingUpdate — the new pod becomes ready before the old one is removed, so there is no gap. |

Java/Spring Boot, and the friendliest of the JVM services to update: the Dockerfile is multi-stage and runs Maven itself, so a plain `docker build` is the whole build. Expect it to be slow the first time after a `pom.xml` change, when the dependency cache layer is invalidated.

## The whole update, in one paste

```bash
REF=main                       # a branch name, or a release tag
cd ~/src/eegfaktura-billing
git fetch --all --tags
git checkout "$REF"
git pull --ff-only 2>/dev/null || echo "(detached at a tag — nothing to pull)"
git log --oneline -1
```

```bash
docker build -t localhost:5000/eegfaktura-billing:dev ~/src/eegfaktura-billing
docker push localhost:5000/eegfaktura-billing:dev
kubectl rollout restart deployment/eegfaktura-billing
kubectl rollout status  deployment/eegfaktura-billing --timeout=300s
```

If any of that fails, the same thing broken into steps is below.

## 1. Fetch

```bash
cd ~/src/eegfaktura-billing
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
docker build -t localhost:5000/eegfaktura-billing:dev ~/src/eegfaktura-billing
```

## 3. Push

```bash
docker push localhost:5000/eegfaktura-billing:dev
```

## 4. Roll out

```bash
kubectl rollout restart deployment/eegfaktura-billing
kubectl rollout status  deployment/eegfaktura-billing --timeout=300s
```

## 5. Verify

```bash
kubectl get pods -l app=eegfaktura-billing
```

```bash
kubectl logs deployment/eegfaktura-billing --tail=40
```

Confirm the running pod is the image you just pushed, not a cached layer:

```bash
kubectl describe pod -l app=eegfaktura-billing | grep -E 'Image:|Image ID:'
```

## Rollback

`kubectl rollout undo` does **not** work here — the previous pod template names
the same mutable `:dev` tag, so it re-pulls the image you are trying to escape.
Tag the build you are replacing *before* you overwrite it:

```bash
SHA=$(git -C ~/src/eegfaktura-billing rev-parse --short HEAD)
docker tag  localhost:5000/eegfaktura-billing:dev localhost:5000/eegfaktura-billing:"$SHA"
docker push localhost:5000/eegfaktura-billing:"$SHA"
```

Then going back is naming it:

```bash
kubectl set image deployment/eegfaktura-billing billing=localhost:5000/eegfaktura-billing:<old-sha>
kubectl rollout status deployment/eegfaktura-billing --timeout=300s
```

## Watch out for

- **`MAIL_HOST` is not optional.** Without it the container exits(1) immediately. The manifest sets it to `eegfaktura-postfix`; upstream does not set it at all, so an update that rewrites the config template can quietly drop it.
- **`-DskipTests` is baked into the Dockerfile.** The image builds even when the test suite is red. If you want the suite to gate the update, run `mvn verify` yourself first — it needs Docker for its Testcontainers Postgres.
- **A `pom.xml` change means a full dependency re-resolve.** The Dockerfile caches `dependency:go-offline` in its own layer, so source-only changes stay fast and version bumps do not.

---

[← all services](README.md)
