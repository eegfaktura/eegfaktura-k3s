# Updating a service from a branch or release

[Steps 08 and 14–18](../docs/) build every image once, from `main`. This page is
the loop you run afterwards: a branch or tag lands in one of the source repos and
you want it in the cluster, without re-reading the step that first built it.

Every service takes the same four moves:

| | | |
|---|---|---|
| **1. Fetch** | `git` in `~/src/<repo>` | new source |
| **2. Rebuild** | `docker build` — plus a pre-step for some | new image, same tag |
| **3. Push** | `docker push localhost:5000/<image>:dev` | registry updated |
| **4. Restart** | `kubectl rollout restart deployment/<name>` | node pulls it |

> [!IMPORTANT]
> **Step 4 is not optional, and it is the one people skip.** Every deployment
> in `k8s/` sets `imagePullPolicy: Always`, but the image *tag* never changes —
> it is always `:dev`. Pushing a new image under an existing tag changes nothing
> in the cluster's spec, so Kubernetes sees no reason to act and the old pod
> keeps running the old bits indefinitely. `rollout restart` is what forces a
> new pod, and `Always` is what makes that new pod fetch the layer you just
> pushed. Together they work; neither works alone.

## Quick reference

| Repo | Image | Deployment | Container | Rebuild is |
|---|---|---|---|---|
| [`eegfaktura-postgresql`](eegfaktura-postgresql.md) | `eegfaktura-postgresql` | `eegfaktura-postgresql` | `postgres` | plain |
| [`eegfaktura-keycloak`](eegfaktura-keycloak.md) | `eegfaktura-keycloak` | `eegfaktura-keycloak` | `keycloak` | plain, slow |
| [`eegfaktura-mosquitto`](eegfaktura-mosquitto.md) | `eegfaktura-mosquitto` | `eegfaktura-mosquitto` | `mosquitto` | plain |
| [`eegfaktura-postfix`](eegfaktura-postfix.md) | `eegfaktura-postfix` | `eegfaktura-postfix` | `postfix` | plain |
| [`eegfaktura-filestore`](eegfaktura-filestore.md) | `eegfaktura-filestore` | `eegfaktura-filestore` | `filestore` | plain |
| [`eegfaktura-backend`](eegfaktura-backend.md) | `vfeeg-backend` | `eegfaktura-backend` | `backend` | plain (codegen is in the Dockerfile) |
| [`eegfaktura-energystore`](eegfaktura-energystore.md) | `energy-store` | `eegfaktura-energystore` | `energystore` | **codegen first** |
| [`eegfaktura-billing`](eegfaktura-billing.md) | `eegfaktura-billing` | `eegfaktura-billing` | `billing` | plain (Maven is in the Dockerfile) |
| [`eegfaktura-admin-backend`](eegfaktura-admin-backend.md) | `eeg-registration-backend` | `eegfaktura-admin-backend` | `admin-backend` | **sbt, then retag** |
| [`eegfaktura-eda-xp`](eegfaktura-eda-xp.md) | `eegfaktura-kep` | `eegfaktura-eda` | `eda` | **sbt, then retag** |
| [`eegfaktura-web`](eegfaktura-web.md) | `vfeeg-web` | `eegfaktura-web` | `web` | **pnpm build first** |
| [`eegfaktura-admin`](eegfaktura-admin.md) | `eeg-registration-frontend` | `eegfaktura-admin-web` | `admin-web` | **npm build first** |

All images live at `localhost:5000/<image>:dev`. **Each repo has its own page**
with the commands already filled in — no placeholders to substitute — including
a single block that does the whole update in one paste. The sections below are
the shared reasoning behind them.

Updating everything at once has its own runbook:
**[complete-update.md](complete-update.md)** — preflight, rollback tagging, all
twelve builds, the rollout in dependency order, and the way back.

## 1. Fetch

```bash
cd ~/src/<repo>
git status --short
```

Deal with anything listed **before** pulling. Two repos dirty themselves as a
side effect of being built:

- `eegfaktura-admin` — `npm install` rewrites `package-lock.json`, deliberately
  ([step 18](../docs/18-frontends.md), known problems #18). Discard it:
  `git checkout package-lock.json`.
- `eegfaktura-web` — a pnpm newer than 9 leaves an untracked
  `pnpm-workspace.yaml` and rewrites `pnpm-lock.yaml`. Same treatment.

```bash
git fetch --all --tags
```

```bash
git switch <branch>        # a branch
git pull --ff-only
```

```bash
git checkout <tag>         # or a release — detached HEAD, which is fine here
```

```bash
git log --oneline -1
```

Note that commit. It is what you tag the image with in
[§5](#5-rollback) if this turns out to need undoing.

## 2. Rebuild

### Plain — postgresql, keycloak, mosquitto, postfix, filestore, backend, billing

Everything these need is inside their Dockerfile.

```bash
docker build -t localhost:5000/<image>:dev ~/src/<repo>
docker push localhost:5000/<image>:dev
```

> [!CAUTION]
> **A new `eegfaktura-postgresql` image does not change a running database.**
> The scripts in `docker-entrypoint-initdb.d/` run exactly once, against an
> empty data directory. If the update adds a table, a user or an extension, the
> rebuilt image will start, log nothing unusual, and serve the *old* schema —
> the PVC already has a cluster in it. Apply such changes by hand with `psql`,
> or accept losing the data and delete the PVC. Also re-read
> [step 10.4](../docs/10-postgres.md#104-hand-the-seeded-schemas-to-their-owners):
> anything the image newly pre-seeds is owned by `postgres` and has to be handed
> to `eegfaktura`, or the services' own migrations fail on it.

> [!CAUTION]
> **A new `eegfaktura-keycloak` image does not re-import the realm.** Keycloak
> imports `realm-export.json` only when its database is empty. An update that
> changes the realm export therefore has no effect on a cluster that has already
> booted once; re-apply the change through the Admin API as in
> [step 12](../docs/12-realm-config.md). The build is also a two-stage
> `kc.sh build` and takes several minutes — it is not hung.

> [!NOTE]
> **`eegfaktura-backend` migrates itself.** `golang-migrate` runs at startup, so
> new migrations in an update apply during the rollout. If one fails, the
> version is left *dirty* and every later start refuses to continue — recovery
> is in [step 15](../docs/15-backend-energystore.md).

### Codegen first — energystore

`energy-store` commits `protoc/excel.pb.go` but not `protoc/masterdata.pb.go`,
and its Dockerfile does not generate. An update that touches a `.proto` file, or
a fresh clone of a branch, needs the generator run by hand first:

```bash
cd ~/src/eegfaktura-energystore
protoc --experimental_allow_proto3_optional=true --proto_path=. --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative ./protoc/*.proto
ls -la protoc/*.pb.go
```

> [!WARNING]
> **`make protoc` silently does nothing** — the target collides with the
> `protoc/` directory and Make considers it up to date
> ([known problems #10](../docs/known-problems.md#10-make-protoc-silently-does-nothing-energystore)).
> Call `protoc` directly as above, or `make -B protoc`.

```bash
docker build -t localhost:5000/energy-store:dev ~/src/eegfaktura-energystore
docker push localhost:5000/energy-store:dev
```

### sbt, then retag — admin-backend, eda-xp

sbt builds the image under **upstream's** name, so it has to be retagged into
your registry. The tag comes from `dockerVersion` in `build.sbt`, and an update
is exactly when that value changes.

```bash
cd ~/src/eegfaktura-admin-backend      # or ~/src/eegfaktura-eda-xp
sbt Docker/publishLocal
```

> [!IMPORTANT]
> **Pick the newest tag, not the first one listed.** Steps 16 and 17 capture the
> tag with `head -1`, which was unambiguous on a first install. After an update
> it is not: both the old and the new build sit in `docker images`, and pushing
> the previous release under `:dev` is a silent no-op update that costs an hour
> to find. Sort by creation time instead, and check the answer against
> `grep -i dockerversion build.sbt`.

```bash
IMG=ghcr.io/vfeeg-development/eeg-registration-backend   # eda-xp: …/eegfaktura-kep
TAG=$(docker images --format '{{.CreatedAt}}\t{{.Tag}}' "$IMG" | grep -v latest | sort -r | head -1 | cut -f2)
echo "$TAG"
```

```bash
docker tag "$IMG:$TAG" localhost:5000/eeg-registration-backend:dev
docker push localhost:5000/eeg-registration-backend:dev
```

For eda-xp the target is `localhost:5000/eegfaktura-kep:dev`, and the deployment
is `eegfaktura-eda` — the names do not match, which is easy to trip over.

### Frontend build first — web, admin

Neither Dockerfile runs npm; both `ADD` a directory that must already exist
([known problems #15](../docs/known-problems.md#15-frontend-dockerfiles-cannot-build-from-a-clean-clone)).

> [!CAUTION]
> **Delete the previous output before rebuilding.** `dist/` and `build/` are
> git-ignored and survive `git switch`, so if the new branch fails to compile,
> `docker build` happily packages the **old** frontend and pushes it as `:dev`.
> The rollout succeeds, the app loads, and you are looking at the previous
> version wondering why your change is missing. Removing the directory first
> turns that into an honest build failure.

Web (pnpm 9 — see [step 18](../docs/18-frontends.md) for why the version is
pinned):

```bash
cd ~/src/eegfaktura-web
rm -rf dist
corepack pnpm@9.12.1 install
corepack pnpm@9.12.1 run build
ls dist
```

```bash
docker build -t localhost:5000/vfeeg-web:dev ~/src/eegfaktura-web
docker push localhost:5000/vfeeg-web:dev
```

Admin (npm, and `npm install` rather than `npm ci` — known problems #18):

```bash
cd ~/src/eegfaktura-admin
rm -rf build
npm install
TSC_COMPILE_ON_ERROR=true npm run build
ls build
```

```bash
docker build -t localhost:5000/eeg-registration-frontend:dev ~/src/eegfaktura-admin
docker push localhost:5000/eeg-registration-frontend:dev
```

## 3. Manifest and configuration changes

An update that adds an environment variable, a port, a volume or a secret needs
the manifest changed too — the image alone will not carry it. Pull this
repository and re-render:

```bash
cd ~/eegfaktura-k3s
git pull --ff-only
for f in k8s/*.yaml; do envsubst '$BASE_DOMAIN $VM_IP' < "$f" > "$MANIFESTS/$(basename "$f")"; done
```

```bash
kubectl apply -f "$MANIFESTS"/<the-changed-file>.yaml
```

Rendering is idempotent and changes nothing in the cluster on its own
([step 4.3](../docs/04-repo-and-settings.md#43-render-the-manifests)). `kubectl
apply` of a changed Deployment restarts the pod by itself — no separate
`rollout restart` needed in that case.

## 4. Restart and verify

```bash
kubectl rollout restart deployment/<name>
kubectl rollout status  deployment/<name> --timeout=300s
```

```bash
kubectl get pods -l app=<name>
kubectl logs deployment/<name> --tail=50
```

Confirm you are actually running the new image rather than a cached layer:

```bash
kubectl describe pod -l app=<name> | grep -E 'Image:|Image ID:'
```

`Image ID` is the digest — compare it against `docker inspect --format
'{{index .RepoDigests 0}}' localhost:5000/<image>:dev` if you have any doubt
that the push landed.

For a user-visible change, finish with the relevant smoke test from the step
that first deployed the service; for the frontends, hard-refresh the browser —
a cached bundle will happily hide a successful rollout.

## 5. Rollback

> [!WARNING]
> **`kubectl rollout undo` does not roll back here.** It restores the previous
> pod template, but that template names `localhost:5000/<image>:dev` — the same
> mutable tag, now pointing at the new image. Kubernetes dutifully pulls the
> thing you were trying to escape. The tag is the problem, not the command.

So push an immutable tag alongside `:dev` for anything you might need to undo:

```bash
SHA=$(git -C ~/src/<repo> rev-parse --short HEAD)
docker tag  localhost:5000/<image>:dev localhost:5000/<image>:"$SHA"
docker push localhost:5000/<image>:"$SHA"
```

Then rolling back is naming the older one explicitly:

```bash
kubectl set image deployment/<name> <container>=localhost:5000/<image>:<old-sha>
kubectl rollout status deployment/<name> --timeout=300s
```

The container names are in the [quick reference](#quick-reference) — they are
not always the deployment name. This is the same reasoning as
[step 20.5](../docs/20-ci-pipeline.md#205-stage-3--deploy), which deploys
`sha-` tags for exactly this reason; doing it by hand here keeps the two
consistent.

## 6. Updating several services at once

[complete-update.md](complete-update.md) is this section made executable — the
loops, the safety net and the verification, in order. The reasoning is here.

Order matters when the update spans repos:

1. **`eegfaktura-postgresql`** — and remember it will not migrate an existing
   database (§2). Do any schema work first, while nothing is writing.
2. **`eegfaktura-keycloak`** — realm and client changes; anything downstream
   authenticates against it.
3. **`eegfaktura-backend`, `eegfaktura-energystore`** — the backend runs its
   migrations during this rollout.
4. **`eegfaktura-admin-backend`, `eegfaktura-billing`, `eegfaktura-eda`**
5. **`eegfaktura-web`, `eegfaktura-admin`** — last, so the UI is never newer
   than the API it calls.

Between 3 and 4, check that nothing is stuck:

```bash
kubectl get pods
```

A `CrashLoopBackOff` here is nearly always a migration or a missing environment
variable, not the image — `kubectl logs deployment/<name> --previous` shows the
run that failed rather than the one currently restarting.

## When it did not take effect

| Symptom | Cause |
|---|---|
| Pod is old, `AGE` unchanged | `rollout restart` was never run — the push alone does nothing |
| New pod, old behaviour | The push did not land, or sbt retagged the previous version (§2) |
| Frontend unchanged | Stale `dist/`/`build/` packaged, or the browser is caching. `rm -rf` and hard-refresh |
| `CrashLoopBackOff` after update | New env var the manifest does not set yet (§3), or a failed migration |
| Schema errors after a postgres image update | Init scripts do not re-run; apply by hand (§2) |
| Login breaks after a keycloak image update | The realm was not re-imported; re-apply via the Admin API ([step 12](../docs/12-realm-config.md)) |
