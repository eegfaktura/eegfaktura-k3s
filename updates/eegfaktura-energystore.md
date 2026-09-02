# Update: `eegfaktura-energystore`

|  |  |
|---|---|
| Source | `~/src/eegfaktura-energystore` |
| Image | `localhost:5000/energy-store:dev` |
| Deployment | `eegfaktura-energystore` |
| Container | `energystore` |
| Manifest | [`k8s/41-energystore.yaml`](../k8s/41-energystore.yaml) |
| First built in | [step 15](../docs/15-backend-energystore.md) |
| Deployed in | [step 15](../docs/15-backend-energystore.md) |
| Rollout | Recreate — the old pod is stopped before the new one starts, so there is a short outage. That is deliberate: the deployment owns a ReadWriteOnce volume and two pods cannot mount it at once. |

The one service with a mandatory pre-build step. It commits `protoc/excel.pb.go` but **not** `protoc/masterdata.pb.go`, and its Dockerfile does not generate — so a clean checkout of any branch fails to compile until you run `protoc` yourself.

## The whole update, in one paste

```bash
REF=main                       # a branch name, or a release tag
cd ~/src/eegfaktura-energystore
git fetch --all --tags
git checkout "$REF"
git pull --ff-only 2>/dev/null || echo "(detached at a tag — nothing to pull)"
git log --oneline -1
```

```bash
cd ~/src/eegfaktura-energystore
protoc --experimental_allow_proto3_optional=true --proto_path=. --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative ./protoc/*.proto
ls -la protoc/*.pb.go
docker build -t localhost:5000/energy-store:dev ~/src/eegfaktura-energystore
docker push localhost:5000/energy-store:dev
kubectl rollout restart deployment/eegfaktura-energystore
kubectl rollout status  deployment/eegfaktura-energystore --timeout=300s
```

If any of that fails, the same thing broken into steps is below.

## 1. Fetch

```bash
cd ~/src/eegfaktura-energystore
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
cd ~/src/eegfaktura-energystore
protoc --experimental_allow_proto3_optional=true --proto_path=. --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative ./protoc/*.proto
ls -la protoc/*.pb.go
```

Then the image:

```bash
docker build -t localhost:5000/energy-store:dev ~/src/eegfaktura-energystore
```

## 3. Push

```bash
docker push localhost:5000/energy-store:dev
```

## 4. Roll out

```bash
kubectl rollout restart deployment/eegfaktura-energystore
kubectl rollout status  deployment/eegfaktura-energystore --timeout=300s
```

## 5. Verify

```bash
kubectl get pods -l app=eegfaktura-energystore
```

```bash
kubectl logs deployment/eegfaktura-energystore --tail=40
```

Confirm the running pod is the image you just pushed, not a cached layer:

```bash
kubectl describe pod -l app=eegfaktura-energystore | grep -E 'Image:|Image ID:'
```

## Rollback

`kubectl rollout undo` does **not** work here — the previous pod template names
the same mutable `:dev` tag, so it re-pulls the image you are trying to escape.
Tag the build you are replacing *before* you overwrite it:

```bash
SHA=$(git -C ~/src/eegfaktura-energystore rev-parse --short HEAD)
docker tag  localhost:5000/energy-store:dev localhost:5000/energy-store:"$SHA"
docker push localhost:5000/energy-store:"$SHA"
```

Then going back is naming it:

```bash
kubectl set image deployment/eegfaktura-energystore energystore=localhost:5000/energy-store:<old-sha>
kubectl rollout status deployment/eegfaktura-energystore --timeout=300s
```

## Watch out for

- **`make protoc` silently does nothing.** The target name collides with the `protoc/` directory and there is no `.PHONY`, so Make reports `'protoc' is up to date` and skips the recipe ([known problems #10](../docs/known-problems.md#10-make-protoc-silently-does-nothing-energystore)). Call `protoc` directly, or `make -B protoc`.
- **Regenerate on every update, not just proto changes.** The generated file is git-ignored, so switching branches can leave you with stubs from the previous version — which compile, and are wrong.
- **It talks gRPC to the backend on 9092.** A proto change on either side needs both images rebuilt, in the same session.

---

[← all services](README.md)
