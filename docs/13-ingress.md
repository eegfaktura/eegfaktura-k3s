# Step 13 — Ingress and routing

The compose stack routes everything through a Caddy container on ports 80/81.
In Kubernetes that container disappears and Traefik does the job — but the
translation has one trap that costs hours if missed.

Manifest: **`k8s/90-ingress.yaml`** — four Middlewares and seven Ingresses.

## 13.1 What Caddy actually does

From `caddy/caddy.conf` in the compose repo:

```
:80  /api/*         -> eegfaktura-backend:9080
     /energystore/* -> eegfaktura-energystore:8080
     /filestore/*   -> eegfaktura-filestore:8080
     /cash/*        -> eegfaktura-billing:8080
     /*             -> eegfaktura-web:8080

:81  /eeg/*  /vfeeg/*  /admin/*  -> eegfaktura-admin-backend:8085
     /*                          -> eegfaktura-admin-web:8080
```

Two hosts replace the two ports:

| Compose | Kubernetes |
|---|---|
| `:8001` (Caddy `:80`) | `app.dev.yourdomain.com` |
| `:8002` (Caddy `:81`) | `admin.dev.yourdomain.com` |

## 13.2 The trap

> [!CAUTION]
> **`handle_path` strips the prefix — Ingress does not.** Caddy's `handle_path`
> *removes* the matched prefix before proxying, so `/api/eeg` reaches the
> backend as `/eeg`. A plain Kubernetes Ingress passes the full path through, so
> the backend receives `/api/eeg` and returns 404. Every prefixed route needs a
> Traefik **StripPrefix** middleware. This is the single most common porting
> mistake here.

And the counter-trap:

> [!WARNING]
> **Not every service wants the prefix stripped.** Filestore mounts its router
> *at* `/filestore`:
> ```python
> app.include_router(filestore.router, prefix=f"/{settings.HTTP_FILE_DL_ENDPOINT}")
> ```
> so stripping breaks it. Its Ingress deliberately has **no** middleware. The
> Caddyfile and the application disagree; trust the running service. See
> [known problems #11](known-problems.md#11-caddyconf-disagrees-with-the-filestore-app).

Establish which form each service serves by asking it directly, rather than
reading the Caddyfile:

```bash
kubectl run curltest --rm -it --restart=Never --image=curlimages/curl -- sh -c \
  "curl -s -o /dev/null -w 'root %{http_code}\n' http://<service>:<port>/; \
   curl -s -o /dev/null -w 'pref %{http_code}\n' http://<service>:<port>/<prefix>/"
```

Use **GET**, not `curl -sI` (HEAD) — several of these services answer 405 to
HEAD regardless of whether the path exists, which makes a working route look
broken. `404` on both means neither is a real route; a `405`, `401` or `200` on
one of them tells you which form the app serves.

## 13.3 Why one Ingress per prefix

The middleware annotation applies to the **whole Ingress object**. Putting
`/api` and `/` in one Ingress would strip `/api` from both. Splitting them gives
each route its own middleware, and Traefik prioritises longer path matches, so
`/api` beats `/` automatically.

> [!NOTE]
> **Middleware reference syntax** is `<namespace>-<name>@kubernetescrd` — the
> namespace prefix is required. The manifest declares the CRD group
> `traefik.io/v1alpha1`; k3s releases before ~2023 used
> `traefik.containo.us/v1alpha1` instead.

## 13.4 Apply

Check the CRD group your Traefik actually installed:

```bash
kubectl get crd | grep -i middleware
```

```bash
kubectl apply -f "$MANIFESTS"/90-ingress.yaml
```

```bash
kubectl get middleware,ingress
```

Four middlewares and seven ingresses.

```bash
curl -s -o /dev/null -w '%{http_code}\n' "https://$APP_HOST/"
```

**404 is the expected answer right now** — no backing Services exist yet. They
come alive one at a time in steps 14–18.

> [!NOTE]
> **404 and 503 mean different things.** Worth learning now, because you will
> see both:
>
> | Response | Meaning |
> |---|---|
> | **404**, bare, no body | The Service does not exist, so Traefik never built the router |
> | **503** `no available server` | The router exists; the Service has no ready endpoints — pod starting or crash-looping |
> | **404 on a prefixed path** *after* the Service exists | The StripPrefix middleware is missing — the backend got `/api/x` instead of `/x` |
> | **404 as JSON**, with a `Server:` header | The request reached the *application*, which does not have that route. Different problem entirely |
> | Connection refused | Never reached Traefik — DNS or firewall |
>
> Always read the `Server` header alongside the status code. A bare 404 is
> Traefik; a JSON 404 is the app.

## Done when

- `kubectl get middleware` lists four (filestore needs none — see 13.2)
- `kubectl get ingress` lists seven
- `https://app.dev.yourdomain.com/` returns 404, from Traefik, with no
  certificate warning

→ [Step 14 — mosquitto, postfix, filestore](14-support-services.md)
