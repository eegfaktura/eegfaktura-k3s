# Step 15 — Backend and Energystore (Go)

The core domain/billing API and the energy time-series store. The backend is the
most involved build in the stack.

Manifests: **`k8s/40-backend.yaml`**, **`k8s/41-energystore.yaml`**

## 15.1 Codegen — and the two repos differ

Both services need generated protobuf bindings, and they handle it **oppositely**:

| Service | Dockerfile runs codegen? | Generated files committed? | You must |
|---|---|---|---|
| `eegfaktura-backend` | **Yes** — `make protoc` + `go generate` | partly (6 of 10) | nothing; `docker build` suffices |
| `eegfaktura-energystore` | **No** — straight to `go build` | only `excel.pb.go` | run codegen **before** building |

Energystore commits `protoc/excel.pb.go` but not `protoc/masterdata.pb.go`, and
its Dockerfile has no codegen step — so a clean clone cannot build:

```
services/apiService.go:16:80: undefined: protobuf.MeteringPoint
services/apiService.go:23:16: undefined: protobuf.NewApiServiceClient
services/apiService.go:28:23: undefined: protobuf.MeteringRequest
```

> [!NOTE]
> **Go has no build hook** — `go build` does not run `go generate`. Generated
> code must either be committed or produced by an explicit step. These two repos
> each chose a different half of that, which is why the fix differs per service.
> [Known problems #8](known-problems.md#8-generated-protobuf-code-is-handled-three-ways).

## 15.2 Build energystore — codegen first

```bash
cd ~/src/eegfaktura-energystore
export PATH="$HOME/go/bin:$PATH"
```

> [!CAUTION]
> **`make protoc` silently does nothing.** The Makefile target is named
> `protoc`, and the repo also contains a **directory** called `protoc/`. Make
> treats the directory as the target, finds it newer than its prerequisites, and
> skips the recipe — there is no `.PHONY: protoc`:
> ```
> make: 'protoc' is up to date.
> ```
> No error, no output, no generated files. See
> [known problems #10](known-problems.md#10-make-protoc-silently-does-nothing-energystore).

Call protoc directly:

```bash
protoc --experimental_allow_proto3_optional=true --proto_path=. --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative ./protoc/*.proto
```

(`make -B protoc` also works, by forcing the recipe.)

```bash
ls -la protoc/*.pb.go
```

Four files: `excel.pb.go`, `excel_grpc.pb.go`, `masterdata.pb.go`,
`masterdata_grpc.pb.go`. The Dockerfile's `COPY . .` then picks them up.

```bash
docker build -t localhost:5000/energy-store:dev ~/src/eegfaktura-energystore
docker push localhost:5000/energy-store:dev
```

## 15.3 Build the backend

Its Dockerfile does the codegen itself, so this is a plain build. It downloads a
Go toolchain, protoc and three plugins, then generates before compiling —
several minutes on first run.

```bash
docker build -t localhost:5000/vfeeg-backend:dev ~/src/eegfaktura-backend
docker push localhost:5000/vfeeg-backend:dev
```

<details>
<summary>Only if you want to build the backend on the host too</summary>

`go build ./...` fails on a clean clone for the same reason. Run codegen first:

```bash
cd ~/src/eegfaktura-backend && make protoc && go generate ./...
```

This is also required before `govulncheck` can build the package graph.

</details>

## 15.4 Create the two ConfigMaps

Compose bind-mounts a config file and a templates directory; both become
ConfigMaps:

```bash
kubectl create configmap backend-config --from-file=config.yaml="$HOME/src/eegfaktura-docker-compose/eegfaktura-backend-config.yaml"
```

```bash
kubectl create configmap backend-templates --from-file="$HOME/src/eegfaktura-docker-compose/templates/"
```

```bash
kubectl get configmap
```

> [!NOTE]
> **`config.yaml`'s ports are authoritative**: `port: 9080` and
> `grpc-provider.port: 9092`. Its database and MQTT values are defaults that the
> environment variables override — only the ports matter. `file-content.templates:
> /opt/storage/public` is why the templates ConfigMap mounts at
> `/opt/storage/public/templates`.

## 15.5 Apply

```bash
kubectl apply -f "$MANIFESTS"/40-backend.yaml -f "$MANIFESTS"/41-energystore.yaml
```

```bash
kubectl rollout status deployment/eegfaktura-backend     --timeout=300s
kubectl rollout status deployment/eegfaktura-energystore --timeout=300s
```

```bash
kubectl logs deployment/eegfaktura-backend --tail=30
```

Expect a database connection, the migrations running to completion, and an MQTT
subscription.

> [!NOTE]
> **The secret is remapped to a filename.** `KEYCLOAK_CONFIG` points at
> `/run/secrets/eegfaktura-keycloak-json`, but the Secret stores the data under
> the key `keycloak.json`. The volume uses an `items:` mapping to expose it under
> the expected filename — matching how compose surfaces a secret under its own
> name. Both manifests also set `automountServiceAccountToken: false`.

<details>
<summary>If the backend logs <code>Dirty database version …</code></summary>

`golang-migrate` recorded a failed migration and refuses to continue. The usual
cause is the schema-ownership problem from
[step 10.4](10-postgres.md#104-hand-the-seeded-schemas-to-their-owners) — the
migration could not create an index on a table owned by `postgres`.

Check the owner:

```bash
kubectl exec -it deployment/eegfaktura-postgresql -- psql -U eegfaktura -d eegfaktura -c "\dn+ base"
```

If it is not `eegfaktura`, reset the schema and restart the backend:

```bash
kubectl exec -i deployment/eegfaktura-postgresql -- psql -U postgres -d eegfaktura <<'SQL'
DROP SCHEMA IF EXISTS base CASCADE;
CREATE SCHEMA base AUTHORIZATION eegfaktura;
SQL
kubectl rollout restart deployment/eegfaktura-backend
```

This destroys application data — harmless now, not after
[step 19](19-bootstrap-and-verify.md).

</details>

<details>
<summary>If the backend panics on <code>context deadline exceeded</code> fetching openid-configuration</summary>

It is trying to reach the URL in `keycloak.json`. Either the `sed` in
[step 9.3](09-namespace-and-secrets.md#93-the-keycloak-client-config) was
skipped, or pods cannot resolve the Keycloak hostname — check
[step 11.4](11-keycloak.md#114-verify-the-issuer-from-inside-a-pod).

```bash
kubectl get secret eegfaktura-keycloak-json -o jsonpath='{.data.keycloak\.json}' | base64 -d | grep -o '"auth-server-url"[^,]*' | sort -u
```

</details>

> [!NOTE]
> The startup line `VFEEG BACKEND is going to listen on 127.0.0.1:9080` is
> **wrong** — the server actually binds `0.0.0.0`. Purely a misleading log
> string; do not chase it.
> [Known problems #14](known-problems.md#14-the-backend-logs-a-bind-address-it-does-not-use).

## 15.6 Probe before trusting the middleware

Filestore proved the Caddyfile is not a reliable guide. Ask the backend directly
which form it serves — GET, not HEAD:

```bash
kubectl run curltest --rm -it --restart=Never --image=curlimages/curl -- sh -c "echo '--- / ---'; curl -s -o /dev/null -w '%{http_code}\n' http://eegfaktura-backend:9080/; echo '--- /api/ ---'; curl -s -o /dev/null -w '%{http_code}\n' http://eegfaktura-backend:9080/api/"
```

| Outcome | Meaning |
|---|---|
| `/` answers, `/api/` 404s | App serves unprefixed — **keep** `strip-api` |
| `/api/` answers, `/` 404s | App serves prefixed — **remove** `strip-api`, as with filestore |
| Both 404 | Neither is a real route; try a known endpoint before concluding |

Then through the ingress:

```bash
curl -s -i "https://$APP_HOST/api/" | head -8
```

`401` or `403` is success — routing works and the endpoint simply wants a token.

## 15.7 Verify the gRPC wiring

Energystore pulls master data from the backend over gRPC on **9092**, so the
Service must publish both ports:

```bash
kubectl get svc eegfaktura-backend -o jsonpath='{.spec.ports}'; echo
```

```bash
kubectl logs deployment/eegfaktura-energystore --tail=30
```

Connection errors mentioning `eegfaktura-backend:9092` mean the gRPC port is
missing from the Service. Exposing only 9080 breaks EEG registration later in a
way that looks like an authentication failure.

## Done when

- Both Pods `Running`
- Backend logs show a database connection and an MQTT subscription
- `/api` returns 401/403 through the ingress, not a bare 404
- Energystore logs show no gRPC connection errors

→ [Step 16 — admin backend](16-admin-backend.md)
