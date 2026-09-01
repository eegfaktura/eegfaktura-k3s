# Step 14 — Mosquitto, Postfix, Filestore

Three supporting services. Mosquitto must exist before the backend and
energystore, which connect to it at startup. Filestore is the first service
behind a **prefixed** ingress route, so it doubles as proof that step 13's
routing works.

Manifests: **`k8s/30-mosquitto.yaml`**, **`k8s/31-postfix.yaml`**,
**`k8s/32-filestore.yaml`**

## 14.1 Build the filestore image

Mosquitto and Postfix were built in [step 08](08-infra-images.md); filestore
still needs building:

```bash
docker build -t localhost:5000/eegfaktura-filestore:dev ~/src/eegfaktura-filestore
docker push localhost:5000/eegfaktura-filestore:dev
```

```bash
curl -s http://localhost:5000/v2/_catalog | jq
```

## 14.2 Apply

```bash
kubectl apply -f "$MANIFESTS"/30-mosquitto.yaml -f "$MANIFESTS"/31-postfix.yaml -f "$MANIFESTS"/32-filestore.yaml
```

```bash
kubectl rollout status deployment/eegfaktura-mosquitto --timeout=120s
kubectl rollout status deployment/eegfaktura-postfix   --timeout=120s
kubectl rollout status deployment/eegfaktura-filestore --timeout=180s
```

```bash
kubectl get pod,svc,pvc
```

> [!NOTE]
> Postfix and Filestore both mount `/run/secrets`, so both set
> `automountServiceAccountToken: false`. Without it the container fails to start
> with `read-only file system` and **empty logs** — see
> [step 10.1](10-postgres.md#101-what-the-manifest-does-and-why).

## 14.3 Verify the prefixed route

The payoff from step 13:

```bash
curl -s -i "https://$APP_HOST/filestore/" | head -8
```

Expected — the application answering, and demanding a token:

```
HTTP/2 401
content-type: application/json
server: uvicorn
www-authenticate: Bearer

{"detail":"Not authenticated"}
```

| Result | Meaning |
|---|---|
| **401** with `server: uvicorn` | Correct — the request reached the app, which requires auth |
| **404**, bare, no body | Traefik has no route: Service missing, or a stale router |
| **404** as JSON from uvicorn | Reached the app, but that path is not registered |
| **503** | Service exists, pod not ready — wait and retry |

Use **GET**, not `curl -sI`. `-I` sends HEAD, which this app rejects with 405,
making a working route look broken.

<details>
<summary>Why filestore has no StripPrefix middleware — measured</summary>

The app mounts its router **at** `/filestore`
(`app.include_router(..., prefix=f"/{HTTP_FILE_DL_ENDPOINT}")`), so the path
must arrive unchanged. Probed against the running service:

```
/            -> 404 Not Found          (no route)
/filestore/  -> 405 Method Not Allowed (route exists, rejects HEAD)
```

`405` is the useful signal: the path is registered. Caddy's
`handle_path /filestore/*` *does* strip, so the compose config and this app
disagree.

</details>

## 14.4 Notes on the configuration

**The mail relay is a placeholder.** `31-postfix.yaml` points at
`smtp.example.com`, so mail is accepted and queued but never delivered. That is
fine here: billing only needs a *reachable* `MAIL_HOST`, and
[step 19](19-bootstrap-and-verify.md) reads the generated registration password
out of the log rather than from an inbox. Point it at a real relay and supply
`eegfaktura-smtp-password` if you want delivery.

**`POSTFIX_MYNETWORK` differs from compose.** Compose leaves it empty; the
manifest sets the k3s pod CIDR so in-cluster senders are trusted. Confirm yours
matches:

```bash
kubectl get node -o jsonpath='{.items[0].spec.podCIDR}'; echo
```

`10.42.0.0/24` from this command is the node's slice of the cluster-wide
`10.42.0.0/16` in the manifest — the wider range is correct.

**Filestore auto-creates storage.** `FILESTORE_CREATE_UNKNOWN_*` are enabled for
first-run convenience; upstream marks them unsuitable for production. The
service treats **any non-empty string as true**, so disabling them means an
empty value, not `"false"`.

## Done when

- All three Pods `Running`
- `https://app.dev.yourdomain.com/filestore/` returns 401 from uvicorn, not a bare 404

→ [Step 15 — backend and energystore](15-backend-energystore.md)
