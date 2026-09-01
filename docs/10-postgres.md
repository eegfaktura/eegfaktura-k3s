# Step 10 — PostgreSQL

One Postgres instance serves both the application and Keycloak. Everything else
depends on it, so it goes first.

Manifest: **`k8s/10-postgres.yaml`** — a PVC, a Deployment and a Service. Read it
before applying; the four decisions worth knowing are below.

## 10.1 What the manifest does and why

**Service name = compose hostname.** Naming the Service `eegfaktura-postgresql`
means every connection string from `docker-compose.yaml`
(`jdbc:postgresql://eegfaktura-postgresql:5432/…`) works unchanged. Every
service in this guide follows that convention.

**`PGDATA` points at a subdirectory.** `local-path` hands over a directory that
is not always empty, and `initdb` refuses to initialise a non-empty one. So
`PGDATA=/var/lib/postgresql/data/pgdata` rather than the mount point itself.

**`strategy: Recreate`.** A `ReadWriteOnce` volume cannot be attached to two
pods, so the default rolling update would deadlock on every change.

**`automountServiceAccountToken: false`** — the one that is not obvious:

> [!CAUTION]
> **`/run/secrets` collides with the service-account token.** Compose puts
> secrets at `/run/secrets`; Kubernetes mounts its service-account token at
> `/var/run/secrets/kubernetes.io/serviceaccount`. In these images `/var/run` is
> a **symlink to `/run`**, so the token mount lands inside the read-only secret
> mount and the container never starts:
> ```
> Error: failed to create containerd task: ... error mounting
> ".../kube-api-access-xxxxx" to rootfs at
> "/var/run/secrets/kubernetes.io/serviceaccount": ...
> mkdirat .../rootfs/run/secrets/kubernetes.io: read-only file system
> ```
> `kubectl logs` is **empty** — the container failed before starting, so the
> evidence is only in `kubectl describe pod`. Declining the token fixes it, and
> none of these services call the Kubernetes API. Five manifests need this.

## 10.2 Apply

Validate against the live API first — `initdb` runs only once, so a typo caught
now is much cheaper than one caught after boot:

```bash
kubectl apply -f "$MANIFESTS"/10-postgres.yaml --dry-run=server
```

```bash
kubectl apply -f "$MANIFESTS"/10-postgres.yaml
```

```bash
kubectl rollout status deployment/eegfaktura-postgresql --timeout=180s
```

```bash
kubectl get pvc,pod
```

## 10.3 Verify both databases exist

```bash
kubectl exec -it deployment/eegfaktura-postgresql -- psql -U postgres -l
```

Both `eegfaktura` and `keycloak` must be listed, reporting `de_DE:UTF8`
collation — recorded but with no locale data behind it, so sorting is byte
order. Expected, and identical to the compose stack
([step 08](08-infra-images.md#83-the-postgres-image-matters-more-than-it-looks)).

```bash
kubectl exec -it deployment/eegfaktura-postgresql -- psql -U eegfaktura -d eegfaktura -c '\dn'
```

Three schemas: `base`, `eda`, `filestore`.

## 10.4 Hand the seeded schemas to their owners

> [!CAUTION]
> **Required before deploying the backend in [step 15](15-backend-energystore.md).**
> The image pre-seeds `base` (9 tables) and `eda` (3 tables) owned by
> **`postgres`**. Both services run their migrations as `eegfaktura` and cannot
> reconcile with tables they do not own — the backend fails on `CREATE INDEX`
> and leaves `golang-migrate` in a *dirty* state; EDA fails with
> `must be owner of table tenantconfig`. Full analysis in
> [known problems #13](known-problems.md#13-the-postgres-image-pre-seeds-schemas-the-services-must-own).

Reset both now, while they are empty and there is nothing to lose:

```bash
kubectl exec -i deployment/eegfaktura-postgresql -- psql -U postgres -d eegfaktura <<'SQL'
DROP SCHEMA IF EXISTS base CASCADE;
CREATE SCHEMA base AUTHORIZATION eegfaktura;
DROP SCHEMA IF EXISTS eda CASCADE;
CREATE SCHEMA eda AUTHORIZATION eegfaktura;
SQL
```

```bash
kubectl exec -it deployment/eegfaktura-postgresql -- psql -U eegfaktura -d eegfaktura -c '\dn'
```

`base` and `eda` must now be owned by `eegfaktura` and be empty. Each service
creates its own tables on first start.

> [!NOTE]
> `filestore` needs no reset. The image creates that schema **empty** and lets
> the service own its own tables — which is why filestore is the only one of the
> three that works untouched, and is good evidence that the other two are a
> packaging mistake rather than a design choice.

## Troubleshooting

| Symptom | Cause |
|---|---|
| Only the `postgres` database exists | Init scripts ran against a pre-existing data directory. Delete the PVC and redeploy — they only run once, on an empty one |
| `password authentication failed` | Trailing newline in a password file. Recheck [9.1](09-namespace-and-secrets.md#91-regenerate-the-passwords) |
| Pod stuck `Pending` | No PV bound. `kubectl get storageclass` must show `local-path` as default |
| `CrashLoopBackOff` with **empty logs** | The `/run/secrets` token collision above. Read `kubectl describe pod`, not `kubectl logs` |

<details>
<summary>Starting over — this destroys all data</summary>

```bash
kubectl delete -f "$MANIFESTS"/10-postgres.yaml
kubectl delete pvc postgres-data
```

Only while there is nothing to lose — i.e. now, not after
[step 19](19-bootstrap-and-verify.md).

</details>

## Done when

- The Pod is `Ready`
- `psql -l` lists `eegfaktura` and `keycloak`
- `base` and `eda` are owned by `eegfaktura` and hold no tables

→ [Step 11 — Keycloak](11-keycloak.md)
