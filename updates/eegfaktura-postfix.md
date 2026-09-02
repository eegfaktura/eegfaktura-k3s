# Update: `eegfaktura-postfix`

|  |  |
|---|---|
| Source | `~/src/eegfaktura-postfix` |
| Image | `localhost:5000/eegfaktura-postfix:dev` |
| Deployment | `eegfaktura-postfix` |
| Container | `postfix` |
| Manifest | [`k8s/31-postfix.yaml`](../k8s/31-postfix.yaml) |
| First built in | [step 08](../docs/08-infra-images.md) |
| Deployed in | [step 14](../docs/14-support-services.md) |
| Rollout | RollingUpdate — the new pod becomes ready before the old one is removed, so there is no gap. |

The outbound mail relay. Billing refuses to start without a mail host, so this one matters more than its size suggests.

## The whole update, in one paste

```bash
REF=main                       # a branch name, or a release tag
cd ~/src/eegfaktura-postfix
git fetch --all --tags
git checkout "$REF"
git pull --ff-only 2>/dev/null || echo "(detached at a tag — nothing to pull)"
git log --oneline -1
```

```bash
docker build -t localhost:5000/eegfaktura-postfix:dev ~/src/eegfaktura-postfix
docker push localhost:5000/eegfaktura-postfix:dev
kubectl rollout restart deployment/eegfaktura-postfix
kubectl rollout status  deployment/eegfaktura-postfix --timeout=300s
```

If any of that fails, the same thing broken into steps is below.

## 1. Fetch

```bash
cd ~/src/eegfaktura-postfix
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
docker build -t localhost:5000/eegfaktura-postfix:dev ~/src/eegfaktura-postfix
```

## 3. Push

```bash
docker push localhost:5000/eegfaktura-postfix:dev
```

## 4. Roll out

```bash
kubectl rollout restart deployment/eegfaktura-postfix
kubectl rollout status  deployment/eegfaktura-postfix --timeout=300s
```

## 5. Verify

```bash
kubectl get pods -l app=eegfaktura-postfix
```

```bash
kubectl logs deployment/eegfaktura-postfix --tail=20
```

Confirm the running pod is the image you just pushed, not a cached layer:

```bash
kubectl describe pod -l app=eegfaktura-postfix | grep -E 'Image:|Image ID:'
```

## Rollback

`kubectl rollout undo` does **not** work here — the previous pod template names
the same mutable `:dev` tag, so it re-pulls the image you are trying to escape.
Tag the build you are replacing *before* you overwrite it:

```bash
SHA=$(git -C ~/src/eegfaktura-postfix rev-parse --short HEAD)
docker tag  localhost:5000/eegfaktura-postfix:dev localhost:5000/eegfaktura-postfix:"$SHA"
docker push localhost:5000/eegfaktura-postfix:"$SHA"
```

Then going back is naming it:

```bash
kubectl set image deployment/eegfaktura-postfix postfix=localhost:5000/eegfaktura-postfix:<old-sha>
kubectl rollout status deployment/eegfaktura-postfix --timeout=300s
```

## Watch out for

- **The queue is not persisted.** There is no volume, so anything still queued when the pod goes away is gone. Restart it when nothing is mid-send.
- **`POSTFIX_MYDOMAIN` comes from the manifest**, rendered from `$BASE_DOMAIN`. If the update changes how the image reads its domain, re-render and re-apply [`k8s/31-postfix.yaml`](../k8s/31-postfix.yaml) as well.
- **Billing depends on it.** If billing was already running, it keeps its connection settings and will simply fail sends during the restart window.

---

[← all services](README.md)
