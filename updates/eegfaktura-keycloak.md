# Update: `eegfaktura-keycloak`

|  |  |
|---|---|
| Source | `~/src/eegfaktura-keycloak` |
| Image | `localhost:5000/eegfaktura-keycloak:dev` |
| Deployment | `eegfaktura-keycloak` |
| Container | `keycloak` |
| Manifest | [`k8s/20-keycloak.yaml`](../k8s/20-keycloak.yaml) |
| First built in | [step 08](../docs/08-infra-images.md) |
| Deployed in | [step 11](../docs/11-keycloak.md) |
| Rollout | RollingUpdate — the new pod becomes ready before the old one is removed, so there is no gap. |

A two-stage build that bakes the custom themes and the realm export into an optimised distribution. It takes several minutes and is not hung. As with Postgres, the interesting part of the image — the realm — is imported only once.

## The whole update, in one paste

```bash
REF=main                       # a branch name, or a release tag
cd ~/src/eegfaktura-keycloak
git fetch --all --tags
git checkout "$REF"
git pull --ff-only 2>/dev/null || echo "(detached at a tag — nothing to pull)"
git log --oneline -1
```

```bash
docker build -t localhost:5000/eegfaktura-keycloak:dev ~/src/eegfaktura-keycloak
docker push localhost:5000/eegfaktura-keycloak:dev
kubectl rollout restart deployment/eegfaktura-keycloak
kubectl rollout status  deployment/eegfaktura-keycloak --timeout=300s
```

If any of that fails, the same thing broken into steps is below.

## 1. Fetch

```bash
cd ~/src/eegfaktura-keycloak
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
docker build -t localhost:5000/eegfaktura-keycloak:dev ~/src/eegfaktura-keycloak
```

## 3. Push

```bash
docker push localhost:5000/eegfaktura-keycloak:dev
```

## 4. Roll out

```bash
kubectl rollout restart deployment/eegfaktura-keycloak
kubectl rollout status  deployment/eegfaktura-keycloak --timeout=300s
```

## 5. Verify

```bash
kubectl get pods -l app=eegfaktura-keycloak
```

```bash
curl -s "https://keycloak.$BASE_DOMAIN/realms/EEGFaktura/.well-known/openid-configuration" | jq -r .issuer
```

Confirm the running pod is the image you just pushed, not a cached layer:

```bash
kubectl describe pod -l app=eegfaktura-keycloak | grep -E 'Image:|Image ID:'
```

## Rollback

`kubectl rollout undo` does **not** work here — the previous pod template names
the same mutable `:dev` tag, so it re-pulls the image you are trying to escape.
Tag the build you are replacing *before* you overwrite it:

```bash
SHA=$(git -C ~/src/eegfaktura-keycloak rev-parse --short HEAD)
docker tag  localhost:5000/eegfaktura-keycloak:dev localhost:5000/eegfaktura-keycloak:"$SHA"
docker push localhost:5000/eegfaktura-keycloak:"$SHA"
```

Then going back is naming it:

```bash
kubectl set image deployment/eegfaktura-keycloak keycloak=localhost:5000/eegfaktura-keycloak:<old-sha>
kubectl rollout status deployment/eegfaktura-keycloak --timeout=300s
```

## Watch out for

- **A changed `realm-export.json` will not be re-imported.** Keycloak imports only into an empty database. Apply realm, client and mapper changes through the Admin API instead — [step 12](../docs/12-realm-config.md) is the pattern, and it is scriptable.
- **The first stage runs `kc.sh build`.** Minutes of apparently idle output is normal; the runtime command is `start --optimized` because of it.
- **Every session dies on restart.** Anyone logged into the platform or admin portal is signed out. Harmless, but tell people first.
- **Check the issuer after the rollout**, from inside the cluster as well as outside — this is the one hostname that must look identical from both sides ([step 11](../docs/11-keycloak.md)).

---

[← all services](README.md)
