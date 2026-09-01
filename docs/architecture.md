# What you are building, and why it is shaped this way

Read this once before starting, or after step 03 when the cluster is up and the
shape starts to matter. The step guides do not depend on it.

## The goal

Take the [eegfaktura](https://github.com/eegfaktura) platform — twelve services
that upstream ships as a `docker-compose` stack — and run it on Kubernetes,
**building every service from source** rather than pulling the published images.

The point is the pipeline: source → image → registry → cluster. Pulling
`ghcr.io/eegfaktura/*` would be faster and would teach you nothing about how the
thing is actually assembled.

**Non-goal:** a production cluster. Single node, no HA, no backups, no secret
manager.

## Target architecture

```
Hypervisor host (x86_64)
└── VM: one Ubuntu/Debian box
    ├── k3s (single node)
    │   ├── containerd · CoreDNS · Traefik (ingress) · local-path (PVCs)
    │   └── namespace: eegfaktura
    ├── registry:2  (localhost:5000)  ← images built on this VM
    └── build toolchain (Go, sbt, Maven, Node, Python, protoc)
```

Three hostnames under a domain you control, all resolving to the VM:

```
keycloak.dev.yourdomain.com   → Keycloak
app.dev.yourdomain.com        → platform UI              (compose :8001)
admin.dev.yourdomain.com      → admin / registration UI  (compose :8002)
```

## The two things that make this non-mechanical

Most of the port is a direct translation. Two things are not, and between them
they account for most of the debugging time this guide saves you.

### 1. Keycloak's hostname is load-bearing

Keycloak stamps its own URL into the `iss` claim of every token, and every
backend fetches signing keys from that same URL. Browser and pods must therefore
agree on **one hostname string** — the addresses behind it may differ, the
string may not.

In compose this is invisible: everything is `localhost` or a compose service
name, and the browser and the containers happen to agree. On Kubernetes they do
not, unless you make them.

Three separate files in the upstream repos hardcode the compose hostname
(`realm-export.json`, `keycloak/keycloak.json`, and the realm's own
`frontendUrl` attribute, which silently overrides the server setting). All three
are rewritten in [step 09](09-namespace-and-secrets.md), and
[step 11](11-keycloak.md) is nothing but verifying the result from both sides.

### 2. HTTPS is a functional requirement, not hardening

The platform frontend computes an S256 PKCE challenge with `crypto.subtle`,
which browsers expose only in a **secure context**. `http://localhost:8001`
qualifies; `http://app.dev.yourdomain.com` does not. Over plain HTTP the app
shows a spinner forever, with no error and no network request.

So TLS comes early — [step 05](05-tls.md), before anything that authenticates —
rather than as a finishing touch. Retrofitting it means moving the realm, both
frontends, the admin backend and the client config together, because a `https://`
page cannot call an `http://` API.

## The compose → Kubernetes mapping

| docker-compose | Kubernetes |
|---|---|
| `services:` | `Deployment` + `Service`, named identically so connection strings work unchanged |
| `secrets:` (file-based) | `Secret`, mounted at the same `/run/secrets/…` paths |
| named `volumes:` | `PersistentVolumeClaim` (local-path) |
| `environment:` | `env`, or a `ConfigMap` for whole files |
| `healthcheck:` | `readinessProbe` |
| `depends_on:` | restart/backoff — deploy order is the guide's job |
| `eegfaktura-proxy` (Caddy) | `Ingress` + Traefik middlewares; the container disappears |
| `ports:` | `Service` ports; only Ingress is exposed externally |

Two mappings have sharp edges, both covered in
[step 13](13-ingress.md): Caddy's `handle_path` strips path prefixes and Ingress
does not, and mounting secrets at `/run/secrets` collides with the
service-account token mount ([step 10](10-postgres.md)).

## Service inventory

| Compose service | Image built | Source repo | Toolchain |
|---|---|---|---|
| `eegfaktura-postgresql` | `eegfaktura-postgresql` | `eegfaktura-postgresql` | Dockerfile |
| `eegfaktura-keycloak` | `eegfaktura-keycloak` | `eegfaktura-keycloak` | Dockerfile (2-stage) |
| `eegfaktura-mosquitto` | `eegfaktura-mosquitto` | `eegfaktura-mosquitto` | Dockerfile |
| `eegfaktura-postfix` | `eegfaktura-postfix` | `eegfaktura-postfix` | Dockerfile |
| `eegfaktura-filestore` | `eegfaktura-filestore` | `eegfaktura-filestore` | Python |
| `eegfaktura-backend` | `vfeeg-backend` | `eegfaktura-backend` | Go 1.25 + protoc |
| `eegfaktura-energystore` | `energy-store` | `eegfaktura-energystore` | Go + protoc (manual) |
| `eegfaktura-admin-backend` | `eeg-registration-backend` | `eegfaktura-admin-backend` | sbt + JDK 17 |
| `eegfaktura-eda` | `eegfaktura-kep` | `eegfaktura-eda-xp` | sbt + JDK 17 |
| `eegfaktura-billing` | `eegfaktura-billing` | `eegfaktura-billing` | Maven + JDK 17 |
| `eegfaktura-web` | `vfeeg-web` | `eegfaktura-web` | Node / Vite / pnpm |
| `eegfaktura-admin-web` | `eeg-registration-frontend` | `eegfaktura-admin` | Node / CRA / npm |
| `eegfaktura-proxy` | — | — | replaced by Ingress |

The build order in steps 08–18 goes easiest-first, so that each new failure has
one likely cause: plain Dockerfiles, then Python, then Go, then Node, then the
JVM services whose builds take longest.

## Sizing

| Resource | Value | Rationale |
|---|---|---|
| vCPU | 8 | sbt and Maven builds are the bottleneck |
| RAM | 24 GB | ~16 GB measured for the running stack, ~1 GB k3s, 4 GB+ for JVM builds |
| Disk | 150 GB | 13 images, the registry, and Go/npm/sbt/Maven caches |

16 GB is a workable floor if you skip the two optional JVM services in
[step 17](17-billing-eda.md) and build one image at a time.

## Known risks

| Risk | Mitigation |
|---|---|
| Keycloak hostname / `iss` mismatch | One name; verify `.well-known/openid-configuration` from browser *and* pod before debugging anything else ([step 11](11-keycloak.md)) |
| Frontend config is rendered at container start | Set env in the Deployment; mounting a file under the webroot does nothing ([step 18](18-frontends.md)) |
| Scala builds are slow and memory-hungry | Build them last; give sbt heap headroom; expect a long first run |
| Upstream image tags can lie | `eeg-registration-backend:v0.2.13` ships a `0.2.10-SNAPSHOT` jar — building from source avoids trusting tags |
| Twenty documented upstream defects | [known-problems.md](known-problems.md) — read it before blaming your own setup |
