# Known problems in the upstream sources

Defects found while porting the eegfaktura stack to Kubernetes. Every entry was
**observed**, not inferred — the evidence column quotes what the tooling actually
reported.

These are problems in the upstream repositories and published images, not in the
Kubernetes port. Each has a workaround applied somewhere in `docs/01`–`20`.

## Summary

| # | Component | Problem | Blocks |
|---|---|---|---|
| 1 | published images | all single-arch `linux/amd64` | Apple Silicon |
| 2 | `eeg-registration-backend` | tag `v0.2.13` ships a `0.2.10-SNAPSHOT` jar | trust in versions |
| 3 | `docker-compose.yaml` | `MAIL_HOST` never defined | billing boot |
| 4 | `keycloak/keycloak.json` | shipped `admin-cli` secret is invalid | auth |
| 5 | `eeg-registration-backend` | ignores `KEYCLOAK_CONFIG_JSON` | EEG registration |
| 6 | `realm-export.json` | hardcodes the compose hostname | token validation |
| 7 | `eegfaktura-postgresql` | declares a locale it cannot provide | sort order |
| 8 | `eegfaktura-backend` | generated code handled three ways | clean-clone build |
| 9 | `eegfaktura-backend` | test scaffolding ships in the binary | binary size, CVEs |
| 10 | `eegfaktura-energystore` | `make protoc` silently does nothing | image build |
| 11 | `caddy.conf` | disagrees with the filestore app | routing |
| 12 | `eegfaktura-backend` | reachable dependency CVEs | security |
| 13 | `eegfaktura-postgresql` | pre-seeds `base` + `eda` tables as the wrong owner | backend **and** EDA migrations |
| 14 | `eegfaktura-backend` | startup log reports the wrong bind address | nothing — misleads debugging |
| 15 | frontend repos | Dockerfiles require an undocumented host build step | image build |
| 16 | `eegfaktura-web` | no `packageManager` pin — newer pnpm drops a security override | reintroduces a CVE |
| 17 | `eegfaktura-web` | `xlsx` has two HIGH CVEs with no npm fix | spreadsheet import |
| 18 | `eegfaktura-admin` | lockfile out of sync + pre-existing TS errors | reproducible builds |
| 19 | `eegfaktura-web` | needs a secure context, and hides the error when it fails | login on any non-localhost HTTP host |
| 20 | `realm-export.json` | no `access_groups` mapper — the backend rejects every logged-in user | the entire app UI |

---

## 1. Published images are `amd64`-only

Every `ghcr.io/eegfaktura/*` image is single-arch `linux/amd64`. Most still pull
on `arm64` with a warning, but `eeg-registration-backend` is published as an
**amd64-only manifest list**, which Docker refuses outright:

```
no matching manifest for linux/arm64/v8 in the manifest list entries
```

**Impact.** The stack cannot run on Apple Silicon without pinning
`platform: linux/amd64` per service and relying on Rosetta emulation.
`caddy` is unaffected — it is genuinely multi-arch.

**Workaround.** `docker-compose.override.yaml` with per-service `platform:`, or
build on `x86_64` (what this project does).

**Real fix.** Publish multi-arch images.

## 2. Image tag does not match its contents

`ghcr.io/eegfaktura/eeg-registration-backend:v0.2.13` contains
`eegfaktura-registration-0.2.10-SNAPSHOT.jar`.

**Impact.** The running artifact is an older snapshot than the tag implies.
Worth suspecting first whenever behaviour contradicts the documentation.

## 3. `MAIL_HOST` is never defined

`docker-compose.yaml` starts `eegfaktura-billing` without `MAIL_HOST`, so Spring
Boot's `MailSenderAutoConfiguration` fails during context startup:

```
PlaceholderResolutionException: Could not resolve placeholder 'MAIL_HOST' in value "${MAIL_HOST}"
```

The container exits(1) on every boot. Upstream has already commented billing out
of the `depends_on` lists of `eegfaktura-web` and `eegfaktura-proxy`, which
suggests the breakage is known.

**Workaround.** Set `MAIL_HOST=eegfaktura-postfix`, `MAIL_PORT=25`.

## 4. The shipped `admin-cli` secret is invalid

The secret committed in `keycloak/keycloak.json` does not match the realm:

```json
{"error":"unauthorized_client","error_description":"Invalid client or Invalid client credentials"}
```

**Workaround.** Regenerate via the Admin API — note that only the `POST` that
regenerates returns the plaintext; a `GET` returns `**********`.

## 5. `eeg-registration-backend` ignores its mounted config

The image's bundled `application.conf` has the line that would load the mounted
file **commented out**:

```hocon
keycloak {
#  configfile = ${?KEYCLOAK_CONFIG_JSON}
  clientId = "admin-cli"
  secret = "P85u55EUB7w6JFjQBxsHDCbdy8TXibDI"   # hardcoded fallback
  secret = ${?KEYCLOAK_ADMIN_CLI_SECRET}         # the only working override
}
```

So `/run/secrets/eegfaktura-keycloak-json` is never read and the service
authenticates with a secret baked into the jar.

**Symptom.** EEG registration fails:

```
Creating Keycloak User failed! jakarta.ws.rs.ProcessingException:
jakarta.ws.rs.NotAuthorizedException: HTTP 401 Unauthorized
```

with, on the Keycloak side:

```
type="CLIENT_LOGIN_ERROR" clientId="admin-cli"
error="invalid_client_credentials" grant_type="client_credentials"
```

**Consequence for the README.** Its step 4 instructs pasting the regenerated
secret into `keycloak/keycloak.json` — a file this service never reads.
**Following the README exactly cannot produce a working stack.**

**Workaround.** Set the undocumented `KEYCLOAK_ADMIN_CLI_SECRET` env var.

## 6. Two config files hardcode the compose hostname

`keycloak/import/realm-export.json` contains:

```json
"attributes": { "frontendUrl": "http://eegfaktura-keycloak:8080" }
```

A realm-level `frontendUrl` **overrides** the server-level `KC_HOSTNAME`, so
Keycloak advertises the compose hostname regardless of the deployment:

```
issuer:  http://eegfaktura-keycloak:8080/realms/EEGFaktura
```

**Why it is hard to spot.** DNS, ingress and the Service can all be correct and
both sides still agree — on the wrong value. Identical wrong answers from two
independent paths point at the *source*, not at resolution.

**Workaround.** Rewrite `frontendUrl` before importing. Re-importing does not
help an existing realm: the startup log shows `Strategy: IGNORE_EXISTING`.

### 6b. The realm export also pins redirect URIs to the compose ports

```json
at.ourproject.vfeeg.app    redirectUris: ["http://localhost:8001/*"]
at.ourproject.vfeeg.admin  redirectUris: ["http://localhost:8002/*"]
```

Any deployment not served on `localhost:8001` / `:8002` fails at login with
**"Invalid parameter: redirect_uri"** — after the issuer and JWKS are already
correct, so it looks like a fresh problem rather than the same hardcoding.

Fixing `frontendUrl` does not help: these are per-client fields. Patch both
clients (see step 15), or rewrite them in the export before first import.

### 6c. `keycloak/keycloak.json` — the same problem, a third file

All four sections carry `"auth-server-url": "http://eegfaktura-keycloak:8080"`.
The backend and energystore read it via `KEYCLOAK_CONFIG` to fetch JWKS, and
panic when the host does not resolve:

```
panic: Get "http://eegfaktura-keycloak:8080/realms/EEGFaktura/.well-known/openid-configuration":
context deadline exceeded (Client.Timeout exceeded while awaiting headers)
```

Fixing `realm-export.json` alone is not enough — the issuer then looks correct
while the services still call the wrong host. Rewrite both.

## 7. Postgres declares a locale it cannot provide

`POSTGRES_INITDB_ARGS: "--locale=de_DE:UTF8"` is accepted and recorded, but the
image ships no locale data:

```
The database cluster will be initialized with locale "de_DE:UTF8".
sh: locale: not found
WARNING:  no usable system locales were found
```

Measured in the running image:

```
SELECT x FROM (VALUES ('Zeta'),('Ölberg'),('Apfel'),('Uhr')) t(x) ORDER BY x;
-->  Apfel, Uhr, Zeta, Ölberg      (byte order — Ö sorts after Z)
-->  Apfel, Ölberg, Uhr, Zeta      (what real de_DE would give)
```

**Impact.** Any database-ordered list — including Excel exports — sorts umlauts
after `Z`. The catalog claims `de_DE:UTF8`, so the setting looks correct.

**Status.** Kept for parity; the compose stack behaves identically.

## 8. Generated protobuf code is handled three ways

In `eegfaktura-backend`, of the ten files `make protoc` produces:

| Files | Treatment |
|---|---|
| `excel`, `mail`, `register` pairs (6) | committed |
| `masterdata` pair | git-ignored |
| `admin` pair | neither — never committed on any branch |

`model/registration_model.go` (from `go generate`) is likewise never committed,
yet is **required to compile** — it defines `model.RegisterEegRequest`, used by
`mqtt/messageBroker.go` and `factory/eegFactory.go`.

**Impact.** A clean clone does not build. Go has no build hook, so `go build`
never runs `go generate`; generated code must be committed or produced by an
explicit step, and this repo does neither consistently.

**Related.** The Dockerfile installs codegen plugins with `@latest`, so two
builds on different days can emit different code — observed as
`SupportPackageIsVersion7` → `9`. Builds are not reproducible.

## 9. Test scaffolding ships in the production binary

`database/test_helper.go` is **not** named `*_test.go`, so Go compiles it into
the binary. It imports `testcontainers-go`, pulling the Docker client, SSH and
tar-extraction code into the shipped artifact — confirmed by 33 `testcontainers`
symbols in the built binary.

**Impact.** Three of the seven vulnerabilities `govulncheck` reports as
reachable in the binary arrive through this path.

**Fix.** Rename to `test_helper_test.go`, or move it to a test-only package.

## 10. `make protoc` silently does nothing (energystore)

`eegfaktura-energystore` commits `protoc/excel.pb.go` but **not**
`protoc/masterdata.pb.go`, and its Dockerfile has no codegen step. Developers
are expected to run `make protoc` first — but that target is broken:

```
make: 'protoc' is up to date.
```

The target is named `protoc`, the repo contains a **directory** named `protoc/`,
and there is no `.PHONY: protoc`. Make treats the directory as the target,
finds it newer than its prerequisites, and skips the recipe. No error, no output,
no files.

**Symptom, two steps later:**

```
services/apiService.go:16:80: undefined: protobuf.MeteringPoint
services/apiService.go:23:16: undefined: protobuf.NewApiServiceClient
```

**Why upstream has not noticed.** Its CI reproduces the protoc command inline
rather than calling `make`.

**Workaround.** Invoke protoc directly, or `make -B protoc`.

## 11. `caddy.conf` disagrees with the filestore app

The proxy strips the prefix:

```
handle_path /filestore/* { reverse_proxy http://eegfaktura-filestore:8080 }
```

but the app mounts its router **at** that prefix:

```python
app.include_router(filestore.router, prefix=f"/{settings.HTTP_FILE_DL_ENDPOINT}")
```

With `HTTP_FILE_DL_ENDPOINT=filestore`, stripping produces `/`, which the app
does not serve. Verified against the running service:

```
/            -> 404 Not Found          (no route)
/filestore/  -> 405 Method Not Allowed (route exists, rejects HEAD)
```

**Lesson.** The Caddyfile is not a reliable description of what each service
expects. Probe each one before translating its route.

## 12. Reachable dependency vulnerabilities (`eegfaktura-backend`)

`osv-scanner` reports 49 findings (32 distinct), but most sit in packages the
code never imports. `govulncheck` against the **built binary** confirms seven as
reachable; three matter:

| Module | Fix | Reached via |
|---|---|---|
| `xuri/excelize/v2` 2.10.1 | → 2.11.0 | `database/excel.go` — the parser consuming user-uploaded `.xlsx` |
| `google.golang.org/grpc` 1.81.0 | → 1.82.1 | `services/grpcServer.go`, `services/mailService.go` |
| `github.com/golang-jwt/jwt` v3.2.2 | migrate to `jwt/v5` | `api/middleware/auth.go` — **no fix exists on v3** |

The excelize one is the most exposed: the core workflow feeds user-supplied
spreadsheets straight into the vulnerable parser.

The remaining four trace to `test_helper.go` (see #9) and disappear if that file
stops shipping.

!!! note "Tooling caveat"
    `osv-scanner` 2.5.1 does not emit reachability data for a `go.mod` scan even
    with `--call-analysis=go`, and its bundled `go1.26` analysis libraries fail
    against a Go 1.27 toolchain. Use `govulncheck`, and note it needs codegen to
    have run first or it cannot build the package graph.


## 13. The Postgres image pre-seeds schemas the services must own

`eegfaktura-postgresql` ships init scripts that create the application schema
before the backend ever starts:

- `11_base_schema.sh` — creates schema `base` **as `postgres`**, granting
  `eegfaktura` rights on the schema only
- `21_base_migration.sql` — creates 9 tables, also as `postgres`, with no
  `ALTER TABLE ... OWNER TO`

The backend then runs golang-migrate as `eegfaktura` against that schema. Two
independent failures result.

**First: ownership.** Tables owned by `postgres` cannot be indexed by
`eegfaktura`, so the migration aborts and marks the database dirty:

```
Dirty database version 20250603103206. Fix and force version.
```

That message is the *guard* on every subsequent boot, not the original error —
which is only visible in the first container, easily lost to a fast crashloop.

**Second, and worse: schema drift.** Even after fixing ownership, the migration
fails because the image's tables are an older shape:

```
migration failed: column "flag" does not exist (column 142) in line 204
CREATE UNIQUE INDEX ... ON "base"."meteringpoint" (..., "flag") WHERE (flag = 1);
```

The migration's own `CREATE TABLE meteringpoint` defines `flag`, but
`CREATE TABLE IF NOT EXISTS` skipped it — the pre-seeded table exists and lacks
the column. Every `IF NOT EXISTS` in that migration silently defers to the older
definition, so the two schemas can never converge.

**Impact.** A fresh install cannot bring the backend up. The two components each
assume they own the schema.

**Fix.** Let the backend own it. After Postgres is up and **before** deploying
the backend:

```sql
DROP SCHEMA IF EXISTS base CASCADE;
CREATE SCHEMA base AUTHORIZATION eegfaktura;
```

The backend's migrations then create everything from scratch, owned correctly.
Its migration set runs well past what the image seeds.

### It affects two of the three seeded schemas

| Schema | Init script creates | Owning service | Result |
|---|---|---|---|
| `base` | schema + **9 tables** | `eegfaktura-backend` (golang-migrate) | breaks |
| `eda` | schema + **3 tables** | `eegfaktura-eda` (Flyway) | breaks |
| `filestore` | schema only, **no tables** | `eegfaktura-filestore` | works |

`filestore` is the accidental control case: because the image creates only an
empty schema, the service creates and owns its own tables and never conflicts.

EDA fails the same way, just with a different migration tool:

```
ERROR: must be owner of table tenantconfig
V20251003180600__extend_type_in_tenantconfig_table.sql
```

Reset both before deploying their services:

```sql
DROP SCHEMA IF EXISTS base CASCADE;
CREATE SCHEMA base AUTHORIZATION eegfaktura;
DROP SCHEMA IF EXISTS eda CASCADE;
CREATE SCHEMA eda AUTHORIZATION eegfaktura;
```

**Real fix upstream.** Have the init scripts create **empty** schemas only —
exactly what `13_filestore_schema.sh` already does — and let each service's
migrations own its tables. The current split is unmaintainable: the image and
the migrations version independently, so their definitions drift apart with no
mechanism to reconcile.


## 14. The backend logs a bind address it does not use

`server.go:213` prints a hardcoded string:

```go
log.Infof("VFEEG BACKEND is going to listen on %s", fmt.Sprintf("127.0.0.1:%d", viper.GetInt("port")))

srv := &http.Server{
    Addr: fmt.Sprintf("0.0.0.0:%d", viper.GetInt("port")),   // the real bind
}
```

So the log claims `127.0.0.1:9080` while the server binds `0.0.0.0:9080`.

**Impact.** None functionally — but in Kubernetes a loopback bind would make the
Service unreachable, so the message sends you chasing a networking fault that
does not exist. The gRPC server logs `:9092` correctly, and the inconsistency
between the two makes the HTTP line look deliberate.

**Fix.** Log `srv.Addr` rather than a literal.


## 15. Frontend Dockerfiles cannot build from a clean clone

Neither frontend Dockerfile runs npm. Both `ADD` a directory that must already
exist on the host:

```
eegfaktura-web    ADD dist  /var/www/html/vfeeg-web/          (Vite output)
eegfaktura-admin  ADD build /var/www/html/registration-web/   (CRA output)
```

So `docker build` on a fresh clone fails:

```
ERROR: failed to compute cache key: "/dist": not found
```

The Makefiles encode the dependency (`docker: build`), so `make docker` works —
but nothing in the Dockerfile signals that a bare `docker build` will not.

**Same family as #10** (energystore's uncommitted protobuf code): the image
build depends on a prior host step that lives outside the Dockerfile. A
multi-stage build would make these self-contained.

**Extra trap.** The two repos use different package managers —
`eegfaktura-web` has `pnpm-lock.yaml`, `eegfaktura-admin` has
`package-lock.json` — so a single build script cannot serve both.


## 16. `eegfaktura-web` does not pin its package manager

`package.json` declares a security override:

```json
"pnpm": { "overrides": { "form-data": "^4.0.4" } }
```

but has **no `packageManager` field**. So `corepack enable` installs the latest
pnpm (11.x observed), and pnpm 11 no longer reads that field:

```
[WARN] The "pnpm" field in package.json is no longer read by pnpm.
       The following keys were ignored: "pnpm.overrides"
```

**Impact.** The override is silently dropped and the build resolves whatever
`form-data` the tree pulls in — defeating a pin that exists precisely to force a
patched version. Nothing fails; the warning is easy to miss among install
output.

The lockfile is `lockfileVersion: '9.0'`, and the repo's own CI pins the exact
version — `pnpm/action-setup@v4` with `version: 9.12.1` in
`.github/workflows/rolling-release.yml`. Use that.

**Secondary symptom.** pnpm 10+ also blocks postinstall scripts by default,
which skips `esbuild`'s native binary and makes `pnpm install` exit 1:

```
[ERR_PNPM_IGNORED_BUILDS] Ignored build scripts: core-js, cypress, esbuild
```

Both problems vanish on pnpm 9.

### It also leaves debris that breaks the correct version

A pnpm 10/11 run **creates a `pnpm-workspace.yaml`** in the repo — a scaffold
for the build scripts it blocked, with placeholder values:

```yaml
allowBuilds:
  core-js: set this to true or false
  cypress: set this to true or false
  esbuild: set this to true or false
```

That file is not tracked upstream, and the same run also **rewrites
`pnpm-lock.yaml`**. Downgrading to pnpm 9 afterwards then fails immediately:

```
ERROR  packages field missing or empty
```

pnpm 9 treats the file as a workspace root and demands a `packages:` field.
Clean up before retrying, and check whether the lockfile was rewritten too:

```bash
git status --short
rm pnpm-workspace.yaml
git checkout pnpm-lock.yaml package.json   # if either shows as modified
```

**Workaround.** Pin the version on every invocation — no root, no shim, and it
cannot drift:

```bash
corepack pnpm@9.12.1 install
corepack pnpm@9.12.1 why form-data      # must be >= 4.0.4
```

A global install works too, if you want a bare `pnpm` on the PATH:

```bash
sudo npm install -g pnpm@9.12.1
```

Note that `corepack prepare pnpm@9.12.1 --activate` on its own does **not**
give you a `pnpm` command — the shims come from `corepack enable`, and running
either under `sudo` records the default for root instead of for you. See
[step 18.2](18-frontends.md).

**Real fix.** Add `"packageManager": "pnpm@9.12.1"` to `package.json` — matching
the version the repo's own CI already pins — so corepack installs the right one
instead of the latest. Longer term, move the overrides to the location current
pnpm reads, so a future upgrade does not silently drop them again.


## 17. Frontend dependency vulnerabilities (`eegfaktura-web`)

`osv-scanner` over `pnpm-lock.yaml` reports **80 distinct findings across 935
packages** (1 critical, 41 high). That headline overstates the exposure: a Vite
build tree-shakes the bundle, so devDependencies never reach a browser.

Splitting by what actually ships:

| Category | Packages | CVEs | Ships? |
|---|---|---|---|
| Direct production deps | 3 | 4 | **yes** |
| Direct dev deps (`vite`) | 1 | 12 | no — dev server only |
| Transitive | 26 | 64 | mostly build tooling |

The single CRITICAL is `tar` 6.2.1 (12 CVEs by itself), pulled in by Cypress and
the Capacitor CLI. It matters for a build pipeline, not for the bundle.

### What genuinely reaches the browser

| Package | Severity | Fix |
|---|---|---|
| `xlsx` 0.18.5 | **2 × HIGH** | **none on npm** |
| `i18next-http-backend` 2.6.2 | MODERATE | 3.0.5 |

**`xlsx` — the one that matters.**

```
CVE-2023-30533  Prototype pollution in SheetJS   fixed in: NO FIX PUBLISHED
CVE-2024-22363  SheetJS ReDoS                    fixed in: NO FIX PUBLISHED
```

`0.18.5` is where the npm-published line ends — SheetJS moved distribution to
their own registry. So remediation is not a version bump: it changes where the
dependency is fetched from, touching the lockfile, CI, and the image build.

!!! danger "The spreadsheet path is vulnerable on both ends"
    The backend's `excelize` (#12) and the frontend's `xlsx` parse **the same
    user-uploaded workbooks**. The Stammdaten/Energiedaten import is this
    application's principal attack surface, and it currently carries reachable
    HIGH-severity parser vulnerabilities server-side *and* client-side — with
    the client half unfixable through normal dependency management.

### What looks alarming but is not

`uuid` 8.3.2 appears with a MODERATE CVE while `package.json` declares
`^9.0.1`. Both versions exist in the tree; the vulnerable one arrives via
`@cypress/request@3.0.5` — test tooling, never bundled. pnpm installs both side
by side rather than hoisting, which is why the lockfile shows the truth.

`package.json` and `pnpm-lock.yaml` are otherwise **in sync**: 66 direct
dependencies declared, 66 locked, no drift.

### Method caveat

This is presence, not reachability. npm has no equivalent of `govulncheck`'s
call-graph analysis, so unlike the backend — where 7 of 32 findings were
confirmed reachable in the built binary — there is no comparable certainty here.
The production/dev split above is the best available proxy.


## 18. `eegfaktura-admin`'s lockfile is out of sync with `package.json`

Upstream documents this in its own CI workflow:

> `npm install` statt `npm ci`: package-lock.json ist im Source-Repo nicht
> synchron mit package.json. `npm ci` erzwingt Sync und failt sofort,
> `npm install` regeneriert den Lock-Stand. Long-term-Fix waere
> `npm install && commit package-lock.json` im Source-Repo.

**Impact.** `npm ci` — the standard for reproducible builds — is unusable. The
lockfile pins the exact floor of every caret range (`@mui/material 5.14.0`,
`@types/react 18.2.14`, `typescript 4.9.5`), and that tree does not type-check:

```
TS2322: Type '(event: FormEvent<HTMLFormElement>) => void' is not assignable
        to type 'FormEventHandler<HTMLDivElement>'
  src/components/common/EditDialog.provider.tsx:48
```

The same `PaperProps={{component: 'form', onSubmit: …}}` pattern appears in
three files, so the build fails on each in turn once the previous is fixed.

**Consequence.** Every build re-resolves dependencies, so no two builds are
guaranteed identical — the opposite of what a lockfile exists to provide. It
also means the build's success depends on what npm resolves *today*.

**Workaround.** Use `npm install`, as CI does. Note CI pins **Node 20**.

**Real fix.** `npm install && git commit package-lock.json` — upstream's own
stated long-term fix, never applied.

### The source also has pre-existing type errors

Separately, the build fails type-checking regardless of the lockfile. Upstream
documents this in the workflow:

> `TSC_COMPILE_ON_ERROR`: der Source hat vorbestehende TS-Typfehler
> (MUI-Dialog-`PaperProps={{component:'form'}}`-Pattern, schon im prod-Image
> v0.2.15 vorhanden). Das JS kompiliert korrekt (babel); nur der strikte
> ForkTsChecker unter CI=true wuerde sie fatal machen.

So the published image was itself built with these errors present. The build
command must be:

```bash
TSC_COMPILE_ON_ERROR=true npm run build
```

**Do not patch the source** — the JS is correct; only the type annotations are
wrong.

### Context: 16 source files were missing entirely

Commit `3ee02be` (2026-06-17) restored ~16 files that had **never been
committed** — they existed only inside the published v0.2.15 image and on one
developer's machine. They were reconstructed from the image's sourcemaps
(`sourcesContent`), and `model/admin.model.ts` was re-derived from its usages.

Until then the repo could not build from source at all. That the type errors
survived into the reconstruction is unsurprising — they were in the original.


## 19. The frontend requires HTTPS, and hides the failure

`oidc-client-ts` computes an S256 PKCE challenge with `crypto.subtle`, which
browsers expose **only in a secure context**. `http://localhost:8001` qualifies
(browsers trust localhost unconditionally); any other host over plain HTTP does
not.

So on `http://app.dev.yourdomain.com` — plain HTTP, not localhost:

```js
window.crypto.subtle   // undefined
```

`signinRedirect()` throws before it can build the authorize URL. The result is a
permanent spinner with **no navigation, no network request, and no error**.

This is invisible in the compose stack, where everything is served from
`localhost` — so the requirement is never hit and never documented.

### Why nothing is displayed

`OidcHandler.tsx` renders in this order:

```jsx
if (auth.isLoading || !auth.isAuthenticated) return <IonSpinner/>;
if (auth.error) { ...show the error... }
```

An unauthenticated user always matches the first branch, so the `auth.error`
branch is **unreachable for exactly the users who need it**. Any authentication
failure — bad config, unreachable Keycloak, missing Web Crypto — presents
identically as a spinner.

**Fix (deployment).** Serve over HTTPS; see [step 05](05-tls.md).

**Fix (upstream).** Check `auth.error` before the loading branch, and detect
`window.isSecureContext` at startup with an explicit message. A one-line
reorder would turn an evening of debugging into a visible error.


## 20. The realm never emits the claim the backend requires

`eegfaktura-backend` authorises every protected route on a group named
`/EEG_ADMIN`, read from an **`access_groups`** claim:

```go
func (ag AccessGroups) IsAdmin() bool {
    for _, s := range ag {
        if s == "/EEG_ADMIN" { return true }
    }
    return false
}
```

`keycloak/import/realm-export.json` contains **zero** occurrences of
`access_groups`. The only group-ish mapper is `groups` in the stock
`microprofile-jwt` scope — an `oidc-usermodel-realm-role-mapper`, which emits
realm *roles*, not group paths.

**Impact.** A correctly registered user with the right groups still gets 401 on
every API call:

```
Unauthorized access with tenant TE100200 - Request has no admin access group
```

The user genuinely holds `/EEG_ADMIN` and `/EEG_OWNER` in Keycloak — the claim
simply never reaches the token. This is easy to misread as a token or issuer
problem, because it appears immediately after login succeeds.

**Fix.** Add a group-membership mapper to `at.ourproject.vfeeg.app`.
`full.path` must be `true`, since the backend compares against the leading-slash
form:

```bash
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "$KC/admin/realms/EEGFaktura/clients/$CID/protocol-mappers/models" \
  -d '{"name":"access_groups","protocol":"openid-connect",
       "protocolMapper":"oidc-group-membership-mapper",
       "config":{"claim.name":"access_groups","full.path":"true",
                 "access.token.claim":"true","id.token.claim":"true",
                 "userinfo.token.claim":"true"}}'
```

Then log out and back in — claims are fixed at token issue time.

**Real fix upstream.** Ship the mapper in `realm-export.json`. As it stands the
exported realm cannot serve the application it belongs to, which suggests the
export was taken from an environment where the mapper was added by hand.
