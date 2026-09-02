# Complete update — every service, in order

One runbook for pulling a new state of the world into a running cluster. It is
the [per-service pages](README.md#quick-reference) executed in dependency order,
with the pre-steps each one needs already inlined.

> [!NOTE]
> **Budget an afternoon.** The two Scala builds and the two Go builds dominate;
> nothing here is faster than the first install was. Nothing is destructive
> either — every block can be re-run.

Order matters, and it is the order the steps deploy in:

```
postgres → keycloak → backend, energystore → admin-backend, billing, eda → web, admin
```

Data first, identity second, APIs third, UI last — so the frontend is never
newer than the API it calls.

## 0. Preflight

```bash
kubectl get pods
```

Start from a healthy cluster. Fixing an unrelated `CrashLoopBackOff` afterwards,
with twelve new images in play, is a much worse afternoon.

```bash
echo "$BASE_DOMAIN / $VM_IP / $MANIFESTS"
kubectl config view --minify -o jsonpath='{..namespace}'; echo
```

The namespace must print `eegfaktura`, or every `kubectl` line below silently
addresses `default`.

## 1. Safety net — tag what is running now

`:dev` is a mutable tag, so `kubectl rollout undo` cannot get you back — it
re-pulls the same tag. Stamp the current images first; this is the only step
that makes the rollback in §7 possible.

```bash
STAMP=$(date +%Y%m%d-%H%M)
echo "rollback stamp: $STAMP"     # write this down
```

```bash
for i in eegfaktura-postgresql eegfaktura-keycloak eegfaktura-mosquitto \
         eegfaktura-postfix eegfaktura-filestore vfeeg-backend energy-store \
         eeg-registration-backend eegfaktura-billing eegfaktura-kep \
         vfeeg-web eeg-registration-frontend; do
  docker pull -q "localhost:5000/$i:dev" >/dev/null 2>&1
  docker tag  "localhost:5000/$i:dev" "localhost:5000/$i:prev-$STAMP" 2>/dev/null &&
  docker push -q "localhost:5000/$i:prev-$STAMP" >/dev/null &&
  echo "tagged  $i:prev-$STAMP" || echo "SKIPPED $i (no :dev image)"
done
```

## 2. Fetch every repo

```bash
REF=main        # a branch, or a release tag if every repo carries the same one
```

```bash
for r in eegfaktura-postgresql eegfaktura-keycloak eegfaktura-mosquitto \
         eegfaktura-postfix eegfaktura-filestore eegfaktura-backend \
         eegfaktura-energystore eegfaktura-admin-backend eegfaktura-billing \
         eegfaktura-eda-xp eegfaktura-web eegfaktura-admin; do
  printf '\n=== %s\n' "$r"
  for lf in package-lock.json pnpm-lock.yaml; do
    git -C ~/src/"$r" checkout -q -- "$lf" 2>/dev/null
  done
  git -C ~/src/"$r" clean -fq -- pnpm-workspace.yaml 2>/dev/null
  git -C ~/src/"$r" fetch -q --all --tags &&
  git -C ~/src/"$r" checkout -q "$REF" &&
  { git -C ~/src/"$r" pull -q --ff-only 2>/dev/null || true; }
  git -C ~/src/"$r" log --oneline -1
done
```

> [!IMPORTANT]
> **The `checkout` and `clean` lines are not tidiness.** `eegfaktura-admin`
> rewrites `package-lock.json` on every `npm install`, and a pnpm newer than 9
> leaves `pnpm-workspace.yaml` behind in `eegfaktura-web` — either one stops the
> pull dead, in a file you never edited. Discarding them first is what makes
> this loop unattended.

> [!TIP]
> **Different refs per repo is the normal case.** Releases are tagged per
> repository, so `REF=main` above is the convenient default, not the usual
> truth. When one repo needs a different ref, fetch it from its own page and
> leave it out of this loop.

## 3. Build and push

Four blocks, because four repos need something before `docker build`. Each is
independent and re-runnable.

### 3a. Everything with a self-contained Dockerfile

postgres, keycloak, mosquitto, postfix, filestore, backend, billing.

```bash
for pair in eegfaktura-postgresql:eegfaktura-postgresql \
            eegfaktura-keycloak:eegfaktura-keycloak \
            eegfaktura-mosquitto:eegfaktura-mosquitto \
            eegfaktura-postfix:eegfaktura-postfix \
            eegfaktura-filestore:eegfaktura-filestore \
            eegfaktura-backend:vfeeg-backend \
            eegfaktura-billing:eegfaktura-billing; do
  repo="${pair%%:*}"; img="${pair##*:}"
  printf '\n=== building %s -> %s\n' "$repo" "$img"
  docker build -t "localhost:5000/$img:dev" ~/src/"$repo" &&
  docker push "localhost:5000/$img:dev" || { echo "FAILED: $repo"; break; }
done
```

### 3b. energystore — generate first

```bash
cd ~/src/eegfaktura-energystore
protoc --experimental_allow_proto3_optional=true --proto_path=. --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative ./protoc/*.proto
ls -la protoc/*.pb.go
```

```bash
docker build -t localhost:5000/energy-store:dev ~/src/eegfaktura-energystore &&
docker push localhost:5000/energy-store:dev
```

### 3c. The two Scala services — build, then retag

```bash
cd ~/src/eegfaktura-admin-backend && sbt Docker/publishLocal
```

```bash
IMG=ghcr.io/vfeeg-development/eeg-registration-backend
TAG=$(docker images --format '{{.CreatedAt}}\t{{.Tag}}' "$IMG" | grep -v latest | sort -r | head -1 | cut -f2)
echo "tag: $TAG"; grep -i dockerversion ~/src/eegfaktura-admin-backend/build.sbt
docker tag "$IMG:$TAG" localhost:5000/eeg-registration-backend:dev &&
docker push localhost:5000/eeg-registration-backend:dev
```

```bash
cd ~/src/eegfaktura-eda-xp && sbt Docker/publishLocal
```

```bash
IMG=ghcr.io/vfeeg-development/eegfaktura-kep
TAG=$(docker images --format '{{.CreatedAt}}\t{{.Tag}}' "$IMG" | grep -v latest | sort -r | head -1 | cut -f2)
echo "tag: $TAG"; grep -i dockerversion ~/src/eegfaktura-eda-xp/build.sbt
docker tag "$IMG:$TAG" localhost:5000/eegfaktura-kep:dev &&
docker push localhost:5000/eegfaktura-kep:dev
```

> [!WARNING]
> **Read the two `tag:` lines.** After an update both the old and the new build
> sit in `docker images`, and picking the wrong one pushes the *previous*
> release under `:dev` — an update that appears to succeed and changes nothing.
> The value must match `dockerVersion` in that repo's `build.sbt`.

### 3d. The two frontends — build, then package

```bash
cd ~/src/eegfaktura-web
rm -rf dist
corepack pnpm@9.12.1 install &&
corepack pnpm@9.12.1 run build &&
ls dist
```

```bash
docker build -t localhost:5000/vfeeg-web:dev ~/src/eegfaktura-web &&
docker push localhost:5000/vfeeg-web:dev
```

```bash
cd ~/src/eegfaktura-admin
rm -rf build
npm install &&
TSC_COMPILE_ON_ERROR=true npm run build &&
ls build
```

```bash
docker build -t localhost:5000/eeg-registration-frontend:dev ~/src/eegfaktura-admin &&
docker push localhost:5000/eeg-registration-frontend:dev
```

> [!CAUTION]
> **The `rm -rf` lines are load-bearing.** `dist/` and `build/` are git-ignored
> and survive `git checkout`. Without removing them, a frontend that fails to
> compile leaves the previous output in place, `docker build` packages it, and
> the rollout succeeds while serving your old code.

## 4. Re-render the manifests

An update that adds an environment variable or a port needs the manifest too —
the image alone will not carry it.

```bash
cd ~/eegfaktura-k3s
git pull --ff-only
for f in k8s/*.yaml; do envsubst '$BASE_DOMAIN $VM_IP' < "$f" > "$MANIFESTS/$(basename "$f")"; done
grep -rn 'BASE_DOMAIN\|VM_IP' "$MANIFESTS"/ && echo "UNEXPANDED — check your shell" || echo "render ok"
```

```bash
kubectl apply -f "$MANIFESTS"/
```

Applying an unchanged manifest is a no-op; applying a changed one restarts that
pod by itself. §5 restarts everything regardless, rather than working out which
ones moved.

## 5. Roll out, in order

```bash
for d in eegfaktura-postgresql eegfaktura-keycloak; do
  kubectl rollout restart deployment/"$d"
  kubectl rollout status  deployment/"$d" --timeout=300s || break
done
```

Both take a real interruption — Postgres because it owns a ReadWriteOnce volume
and cannot run two pods at once, Keycloak because every session is invalidated.
Let them settle before continuing.

```bash
for d in eegfaktura-mosquitto eegfaktura-postfix eegfaktura-filestore \
         eegfaktura-backend eegfaktura-energystore; do
  kubectl rollout restart deployment/"$d"
  kubectl rollout status  deployment/"$d" --timeout=300s || break
done
```

The backend runs its database migrations during that rollout. If it stalls here,
read its log before restarting anything else — a failed migration leaves
`golang-migrate` dirty, and repeated restarts will not clear it.

```bash
for d in eegfaktura-admin-backend eegfaktura-billing eegfaktura-eda \
         eegfaktura-web eegfaktura-admin-web; do
  kubectl rollout restart deployment/"$d"
  kubectl rollout status  deployment/"$d" --timeout=300s || break
done
```

## 6. Verify

```bash
kubectl get pods
```

Twelve pods `Running`, none with a restart count climbing.

```bash
for d in eegfaktura-postgresql eegfaktura-keycloak eegfaktura-mosquitto \
         eegfaktura-postfix eegfaktura-filestore eegfaktura-backend \
         eegfaktura-energystore eegfaktura-admin-backend eegfaktura-billing \
         eegfaktura-eda eegfaktura-web eegfaktura-admin-web; do
  printf '%-30s %s\n' "$d" "$(kubectl get pod -l app=$d -o jsonpath='{.items[0].status.containerStatuses[0].imageID}' 2>/dev/null | cut -c1-60)"
done
```

Every line must carry a digest. A service still showing the digest it had before
the update means its push did not land.

```bash
curl -s "https://keycloak.$BASE_DOMAIN/realms/EEGFaktura/.well-known/openid-configuration" | jq -r .issuer
```

```bash
curl -sI "https://app.$BASE_DOMAIN"   | head -1
curl -sI "https://admin.$BASE_DOMAIN" | head -1
```

Then the end-to-end check that actually proves the chain, from
[step 19.5](../docs/19-bootstrap-and-verify.md#195-import-the-master-data):

```bash
kubectl exec -it deployment/eegfaktura-postgresql -- psql -U eegfaktura -d eegfaktura \
  -c 'select count(*) as participants from base.participant;'
```

Log in at `https://app.$BASE_DOMAIN` with a **hard refresh** — a cached bundle
will show you the old frontend over a perfectly good rollout.

## 7. Rollback

Using the stamp from §1:

```bash
STAMP=<the-stamp-you-wrote-down>
```

One service:

```bash
kubectl set image deployment/eegfaktura-backend backend=localhost:5000/vfeeg-backend:prev-"$STAMP"
kubectl rollout status deployment/eegfaktura-backend --timeout=300s
```

Everything, in reverse order — UI first, so it never outlives its API:

```bash
for t in eegfaktura-admin-web:admin-web:eeg-registration-frontend \
         eegfaktura-web:web:vfeeg-web \
         eegfaktura-eda:eda:eegfaktura-kep \
         eegfaktura-billing:billing:eegfaktura-billing \
         eegfaktura-admin-backend:admin-backend:eeg-registration-backend \
         eegfaktura-energystore:energystore:energy-store \
         eegfaktura-backend:backend:vfeeg-backend \
         eegfaktura-filestore:filestore:eegfaktura-filestore \
         eegfaktura-postfix:postfix:eegfaktura-postfix \
         eegfaktura-mosquitto:mosquitto:eegfaktura-mosquitto \
         eegfaktura-keycloak:keycloak:eegfaktura-keycloak \
         eegfaktura-postgresql:postgres:eegfaktura-postgresql; do
  d="${t%%:*}"; rest="${t#*:}"; c="${rest%%:*}"; img="${rest##*:}"
  kubectl set image deployment/"$d" "$c"=localhost:5000/"$img":prev-"$STAMP" &&
  kubectl rollout status deployment/"$d" --timeout=300s || break
done
```

> [!CAUTION]
> **A rollback does not undo a migration.** The backend's `golang-migrate` and
> Keycloak's own schema changes have already been applied to the database, and
> an older image may not run against a newer schema. Rolling the *code* back is
> easy; rolling the *data* back is a restore. Weigh that before updating
> anything that migrates.

## When it did not take effect

| Symptom | Cause |
|---|---|
| Pod age unchanged | The restart in §5 never ran for that deployment |
| New pod, old behaviour | The push failed, or §3c picked the previous sbt tag |
| Frontend unchanged | Stale `dist/`/`build/` was packaged, or the browser is caching |
| `CrashLoopBackOff` after §5 | An env var the manifest does not set yet — re-run §4 |
| Backend stuck on `Dirty database version` | A migration failed; see [step 15](../docs/15-backend-energystore.md) |
| Login broken | A realm change a new Keycloak image does not re-import — [step 12](../docs/12-realm-config.md) |
| Schema errors | Postgres init scripts do not re-run on an existing volume |

---

[← all services](README.md)
