# Update: `eegfaktura-mosquitto`

|  |  |
|---|---|
| Source | `~/src/eegfaktura-mosquitto` |
| Image | `localhost:5000/eegfaktura-mosquitto:dev` |
| Deployment | `eegfaktura-mosquitto` |
| Container | `mosquitto` |
| Manifest | [`k8s/30-mosquitto.yaml`](../k8s/30-mosquitto.yaml) |
| First built in | [step 08](../docs/08-infra-images.md) |
| Deployed in | [step 14](../docs/14-support-services.md) |
| Rollout | Recreate — the old pod is stopped before the new one starts, so there is a short outage. That is deliberate: the deployment owns a ReadWriteOnce volume and two pods cannot mount it at once. |

The MQTT broker between backend, energystore and EDA. Small image, fast rebuild, and the only thing to think about is who is connected to it.

## The whole update, in one paste

```bash
REF=main                       # a branch name, or a release tag
cd ~/src/eegfaktura-mosquitto
git fetch --all --tags
git checkout "$REF"
git pull --ff-only 2>/dev/null || echo "(detached at a tag — nothing to pull)"
git log --oneline -1
```

```bash
docker build -t localhost:5000/eegfaktura-mosquitto:dev ~/src/eegfaktura-mosquitto
docker push localhost:5000/eegfaktura-mosquitto:dev
kubectl rollout restart deployment/eegfaktura-mosquitto
kubectl rollout status  deployment/eegfaktura-mosquitto --timeout=300s
```

If any of that fails, the same thing broken into steps is below.

## 1. Fetch

```bash
cd ~/src/eegfaktura-mosquitto
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
docker build -t localhost:5000/eegfaktura-mosquitto:dev ~/src/eegfaktura-mosquitto
```

## 3. Push

```bash
docker push localhost:5000/eegfaktura-mosquitto:dev
```

## 4. Roll out

```bash
kubectl rollout restart deployment/eegfaktura-mosquitto
kubectl rollout status  deployment/eegfaktura-mosquitto --timeout=300s
```

## 5. Verify

```bash
kubectl get pods -l app=eegfaktura-mosquitto
```

```bash
kubectl logs deployment/eegfaktura-mosquitto --tail=20
```

Confirm the running pod is the image you just pushed, not a cached layer:

```bash
kubectl describe pod -l app=eegfaktura-mosquitto | grep -E 'Image:|Image ID:'
```

## Rollback

`kubectl rollout undo` does **not** work here — the previous pod template names
the same mutable `:dev` tag, so it re-pulls the image you are trying to escape.
Tag the build you are replacing *before* you overwrite it:

```bash
SHA=$(git -C ~/src/eegfaktura-mosquitto rev-parse --short HEAD)
docker tag  localhost:5000/eegfaktura-mosquitto:dev localhost:5000/eegfaktura-mosquitto:"$SHA"
docker push localhost:5000/eegfaktura-mosquitto:"$SHA"
```

Then going back is naming it:

```bash
kubectl set image deployment/eegfaktura-mosquitto mosquitto=localhost:5000/eegfaktura-mosquitto:<old-sha>
kubectl rollout status deployment/eegfaktura-mosquitto --timeout=300s
```

## Watch out for

- **Three services lose their broker for a few seconds.** Backend, energystore and EDA all reconnect on their own; if one does not, restart it rather than debugging the broker.
- **Config lives in the image.** A changed `mosquitto.conf` needs this rebuild — there is no ConfigMap to patch instead.

---

[← all services](README.md)
