# eegfaktura on k3s — a build-it-yourself cookbook

Run the [eegfaktura](https://github.com/eegfaktura) energy-community platform on
a single-node Kubernetes cluster, **building all twelve services from source**.

Upstream ships a `docker-compose` stack and keeps its Kubernetes deployment in a
private repository. This is an independent port: 20 steps, 15 manifests, and a
log of the [20 upstream defects](docs/known-problems.md) you will otherwise
rediscover one at a time — four of which mean a fresh install cannot work by
following the official README.

Everything here was executed on a real cluster, not written from the
docker-compose file. Where a step says a command prints something, that is what
it printed.

> **Not a production deployment.** Single node, no HA, no backups, no secret
> manager. It is a development and learning environment.

## What you need

- A VM: 8 vCPU / 24 GB RAM / 150 GB disk, x86_64, Ubuntu 24.04 or Debian 12
  (16 GB works if you skip the two optional JVM services)
- **A domain you control** — not necessarily public, but you must be able to add
  DNS records, because a real TLS certificate is required, not optional
- A few hours. The Scala and Go builds are not fast.

## Start here

| # | Step | Covers |
|---|---|---|
| 01 | [The VM](docs/01-vm.md) | sizing, static IP, the two settings everything else uses |
| 02 | [Domain and DNS](docs/02-domain-and-dns.md) | three hostnames, resolvable from browser **and** pods |
| 03 | [k3s](docs/03-k3s.md) | cluster, kubeconfig, ingress smoke test |
| 04 | [The repo and your settings](docs/04-repo-and-settings.md) | clone, substitute your domain, namespace |
| 05 | [TLS](docs/05-tls.md) | **required** — wildcard certificate via DNS-01 |
| 06 | [Registry](docs/06-registry.md) | Docker as a build tool + a local registry |
| 07 | [Toolchains](docs/07-toolchains.md) | Go, protoc, Node, JDK 17, sbt, Maven, Python |
| 08 | [Infrastructure images](docs/08-infra-images.md) | postgres, mosquitto, postfix, keycloak |
| 09 | [Namespace, secrets, config](docs/09-namespace-and-secrets.md) | compose secrets → Kubernetes Secrets |
| 10 | [PostgreSQL](docs/10-postgres.md) | one server, two databases, **schema ownership** |
| 11 | [Keycloak](docs/11-keycloak.md) | **the issuer step** — verify from both sides |
| 12 | [Realm configuration](docs/12-realm-config.md) | admin-cli secret, redirect URIs, `access_groups` mapper |
| 13 | [Ingress and routing](docs/13-ingress.md) | Caddy → Traefik, **StripPrefix middlewares** |
| 14 | [Support services](docs/14-support-services.md) | mosquitto, postfix, filestore |
| 15 | [Backend and energystore](docs/15-backend-energystore.md) | Go services, codegen, gRPC :9092 |
| 16 | [Admin backend](docs/16-admin-backend.md) | Scala/sbt, `KEYCLOAK_ADMIN_CLI_SECRET` |
| 17 | [Billing and EDA](docs/17-billing-eda.md) | optional JVM services, `MAIL_HOST` |
| 18 | [Frontends](docs/18-frontends.md) | React apps, env-rendered Keycloak config |
| 19 | [Bootstrap and verify](docs/19-bootstrap-and-verify.md) | manager user, register an EEG, import master data |
| 20 | [Build pipeline](docs/20-ci-pipeline.md) | recreate the GitHub Actions flow |

Background, if you want it: [docs/architecture.md](docs/architecture.md) —
what is being built and why it is shaped this way.

### The five things that will bite you

1. **HTTPS is functional, not cosmetic** (05) — the frontend needs a secure
   context for Web Crypto. Over plain HTTP it spins forever with no error at all.
2. **Schema ownership** (10) — the Postgres image seeds tables as the wrong
   user, and two services' migrations then cannot run.
3. **Keycloak's hostname** (11) — one name, resolvable from browser and pods;
   `iss` must match the JWKS URL for both.
4. **`access_groups`** (12) — the realm never emits the claim the backend
   requires, so login succeeds and then every API call returns 401.
5. **StripPrefix** (13) — Caddy strips path prefixes; Kubernetes Ingress does
   not. Except for filestore, which wants the prefix kept.

## How to read these docs

**Copy only what is between the ``` fences, never the fences themselves.**
Backticks start command substitution in bash, so pasting a ```` ```bash ```` line
makes the shell swallow everything after it — the commands appear to run and
produce no output.

Otherwise:

| You see | It means |
|---|---|
| A plain code block | A step to run |
| A `> [!NOTE]` / `> [!WARNING]` block | Context or a trap. Read it; nothing to run |
| A collapsed **▸ details** section | Conditional — recovery, diagnostics, or an alternative. Open it only if its summary describes your situation |

Every step ends with a **Done when** checklist. They are load-bearing: a missed
check surfaces four steps later as something that looks unrelated.

Commands use shell variables (`$BASE_DOMAIN`, `$KC`, `$K8S`, …) set in
[step 01](docs/01-vm.md#13-record-your-settings), so they can be pasted
unchanged. Prose and browser URLs spell out the placeholder
`dev.yourdomain.com` — substitute your own.

## Layout

```
docs/    the 20 step guides, the architecture note, and known-problems.md
k8s/     Kubernetes manifests, numbered like the upstream platform repo
```

The manifests are real files, applied straight from the clone on the VM —
nothing in this guide asks you to copy YAML out of markdown. Their numbers
follow upstream's convention and are independent of the step numbers; the map
is in [step 04](docs/04-repo-and-settings.md#42-what-is-in-here).

## Known problems

[docs/known-problems.md](docs/known-problems.md) documents 20 defects in the
upstream repositories and published images, each with the evidence that proved
it. Four of them block a fresh install outright:

- `eegfaktura-energystore` cannot build from a clean clone — generated protobuf
  code is neither committed nor generated, and `make protoc` silently no-ops
- Both frontend Dockerfiles require an undocumented host build step
- The Postgres image seeds `base` and `eda` tables owned by `postgres`, so the
  backend's and EDA's migrations cannot run
- The realm export contains no `access_groups` mapper, so the backend rejects
  every authenticated user

Read it before assuming a failure is yours.

## Related

| Repo | Role |
|---|---|
| [`eegfaktura/eegfaktura-docker-compose`](https://github.com/eegfaktura/eegfaktura-docker-compose) | the stack this ports from — and the source of truth for every env var and config file |
| [`eegfaktura/eegfaktura-docs`](https://github.com/eegfaktura/eegfaktura-docs) | upstream architecture and service documentation |

## Licence

The manifests and documentation here are independent work. The eegfaktura
services themselves are licensed by their own repositories.
