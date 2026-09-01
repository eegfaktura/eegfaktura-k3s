# Step 18 — Frontends (React)

Two static React apps. The builds are unremarkable; the configuration has one
trap, and each repo has its own packaging quirk.

Manifests: **`k8s/70-web.yaml`**, **`k8s/71-admin-web.yaml`**

## 18.1 How these images get their Keycloak URL

> [!CAUTION]
> **Do not mount a config file — it will be ignored.** Both images serve their
> assets with Caddy out of `/srv`, and render `/config/keycloak-config.json`
> **from a template at container start** using environment variables. Mounting a
> file under the webroot does not override it: Caddy serves `/config/...` from
> the template, not from disk. With the variables unset, both apps fall back to
> `http://localhost:7080` and every login fails.

So configuration is purely `env` in the Deployment — which is what the manifests
do, differing only in `KEYCLOAK_CLIENT_ID`.

## 18.2 Build — npm first, then docker

> [!CAUTION]
> **The Dockerfiles only package a pre-built frontend.** Neither runs npm. They
> `ADD` a directory that must already exist:
> ```
> eegfaktura-web    ADD dist  /var/www/html/vfeeg-web/          (Vite output)
> eegfaktura-admin  ADD build /var/www/html/registration-web/   (CRA output)
> ```
> `docker build` on a clean clone fails with
> `ERROR: failed to compute cache key: "/dist": not found`. Each Makefile
> declares `docker: build`, so `make docker` runs the npm build first — a bare
> `docker build` skips it.
> [Known problems #15](known-problems.md#15-frontend-dockerfiles-cannot-build-from-a-clean-clone).

The two repos use **different package managers**: `eegfaktura-web` ships a
`pnpm-lock.yaml`, `eegfaktura-admin` a `package-lock.json`.

### Web — pnpm, output in `dist/`

```bash
cd ~/src/eegfaktura-web
```

> [!WARNING]
> **Use pnpm 9.** `eegfaktura-web` pins `"form-data": "^4.0.4"` via
> `pnpm.overrides` to close a CVE, but declares no `packageManager`. Corepack
> therefore installs the latest pnpm (11.x observed), which **ignores that
> field** — silently dropping the pin and reintroducing the vulnerability.
> pnpm 10+ additionally blocks postinstall scripts, skipping `esbuild`'s native
> binary and failing the build with `ERR_PNPM_IGNORED_BUILDS`. The repo's CI
> pins `9.12.1`; match it.
> [Known problems #16](known-problems.md#16-eegfaktura-web-does-not-pin-its-package-manager).

```bash
sudo corepack prepare pnpm@9.12.1 --activate
pnpm --version
```

<details>
<summary>If pnpm is already installed and this collides</summary>

`corepack enable` symlinks into `/usr/bin` and needs root:

```
Internal Error: EACCES: permission denied, symlink ... -> '/usr/bin/pnpm'
```

and installing pnpm through npm afterwards collides with corepack's shims:

```
npm error EEXIST: file already exists
npm error File exists: /usr/bin/pnpx
```

`corepack prepare … --activate` above avoids both. To use npm's global install
instead, clear the shims first:

```bash
sudo corepack disable && sudo rm -f /usr/bin/pnpm /usr/bin/pnpx
sudo npm install -g pnpm@9.12.1
```

</details>

<details>
<summary>If a newer pnpm already ran in this checkout</summary>

It creates an untracked `pnpm-workspace.yaml` (an `allowBuilds` scaffold) **and
rewrites `pnpm-lock.yaml`**. The stray file then breaks pnpm 9 with
`ERROR packages field missing or empty`:

```bash
git status --short
rm -f pnpm-workspace.yaml
git checkout pnpm-lock.yaml package.json    # only if shown as modified
```

</details>

> [!NOTE]
> Do **not** substitute `npm install` here — it ignores `pnpm-lock.yaml` and
> re-resolves from `package.json`, producing a different dependency tree than the
> lockfile pins.

```bash
pnpm install
pnpm run build
ls dist
```

```bash
docker build -t localhost:5000/vfeeg-web:dev ~/src/eegfaktura-web
docker push localhost:5000/vfeeg-web:dev
```

### Admin — npm, output in `build/`

```bash
cd ~/src/eegfaktura-admin
```

> [!WARNING]
> **Use `npm install`, not `npm ci`.** This repo's `package-lock.json` is not in
> sync with `package.json` — upstream's own CI says so and uses `npm install`
> for that reason. `npm ci` installs strictly from the lockfile, pinning the
> floor of every caret range (`@mui/material 5.14.0`, `@types/react 18.2.14`),
> and that tree fails to type-check. `npm install` re-resolves and **will**
> modify `package-lock.json`; that is expected here.
> [Known problems #18](known-problems.md#18-eegfaktura-admins-lockfile-is-out-of-sync-with-packagejson).

```bash
npm install
```

> [!WARNING]
> **`TSC_COMPILE_ON_ERROR=true` is required.** The source carries pre-existing
> TypeScript errors — the MUI `PaperProps={{component: 'form'}}` pattern in three
> components, present since the published v0.2.15 image. Babel strips types and
> emits correct JS; only the strict ForkTsChecker objects, and upstream's CI sets
> this variable and builds through them deliberately:
> ```
> TS2322: Type '(event: FormEvent<HTMLFormElement>) => void' is not assignable
>         to type 'FormEventHandler<HTMLDivElement>'
> ```
> Do **not** patch the source — it works as intended.

```bash
TSC_COMPILE_ON_ERROR=true npm run build
ls build
```

`build/` must contain `index.html` **and** a `static/` directory. If it holds
only `favicon.ico`, `manifest.json` and `robots.txt`, the compile failed and
that is just `public/` copied across — an image built from it looks fine and
serves a broken frontend.

```bash
docker build -t localhost:5000/eeg-registration-frontend:dev ~/src/eegfaktura-admin
docker push localhost:5000/eeg-registration-frontend:dev
```

> [!CAUTION]
> **Always pass the build context explicitly — never `.`.** Both Dockerfiles
> `ADD` a relative path, so the context must be the repo root. Passing it as an
> argument does that regardless of your working directory. Building `vfeeg-web`
> from the admin directory tags the **admin** app as `vfeeg-web`, and both
> hostnames then serve the same frontend — with only the env-rendered
> `keycloak-config.json` differing, which makes the routing look correct while
> the wrong app is served.

```bash
cd ~
```

## 18.3 Apply

Confirm both images reached the registry first — applying before they exist
gives `ErrImagePull`, which reads as a cluster fault rather than a missed push:

```bash
curl -s http://localhost:5000/v2/_catalog | jq
```

`vfeeg-web` and `eeg-registration-frontend` must both be listed.

```bash
kubectl apply -f "$MANIFESTS"/70-web.yaml -f "$MANIFESTS"/71-admin-web.yaml
```

```bash
kubectl rollout status deployment/eegfaktura-web       --timeout=180s
kubectl rollout status deployment/eegfaktura-admin-web --timeout=180s
```

```bash
kubectl get pods
```

Wait for both rollouts before checking anything: a pod listed seconds after
`apply` is still pulling, so `0/1` at that moment means nothing.

## 18.4 Verify the two hosts serve different apps

```bash
kubectl exec deployment/eegfaktura-web       -- ls /var/www/html/
kubectl exec deployment/eegfaktura-admin-web -- ls /var/www/html/
```

`vfeeg-web` and `registration-web` respectively. Then from outside:

```bash
for h in "$APP_HOST" "$ADMIN_HOST"; do echo "--- $h"; curl -s "https://$h/" | grep -oE "<title>[^<]*</title>|assets/[a-zA-Z0-9.-]*\.js|/static/js/[a-z0-9.]*\.js" | head -2; done
```

The app host is Vite-built (`assets/…`), the admin host CRA-built
(`/static/js/main.….js`). **Identical output from both means one image was built
from the wrong directory.**

## 18.5 Verify the rendered config

The check that catches the trap in 18.1:

```bash
curl -s "https://$APP_HOST/config/keycloak-config.json"   | jq
curl -s "https://$ADMIN_HOST/config/keycloak-config.json" | jq
```

Each must show your `https://keycloak.<your-domain>` URL and the correct
`resource` — `at.ourproject.vfeeg.app` and `at.ourproject.vfeeg.admin`
respectively. `localhost:7080` means the environment variables did not reach the
container.

Then open both in a browser:

- `https://app.dev.yourdomain.com`
- `https://admin.dev.yourdomain.com`

Both must redirect to the Keycloak login page on
`keycloak.dev.yourdomain.com` — confirming the issuer chain end to end.

<details>
<summary>If a page shows a spinner and never navigates</summary>

The app failed before it could build the authorize URL, and renders its loading
branch instead of its error branch, so nothing is displayed. Check, in order:

1. **Are you on `https://`?** In the browser console, `window.crypto.subtle`
   must be an object. If it is `undefined`, you are on plain HTTP — see
   [step 05](05-tls.md).
2. **Is the config right?** The `keycloak-config.json` check above.
3. **Are the redirect URIs registered?** A `redirect_uri` rejection normally
   shows a Keycloak error page, but check
   [step 12.3](12-realm-config.md#123-register-your-redirect-uris) anyway.

[Known problems #19](known-problems.md#19-the-frontend-requires-https-and-hides-the-failure)
explains why no error is shown.

</details>

## Done when

- Each host serves a different app, confirmed by its asset paths
- Both `config/keycloak-config.json` endpoints show your `https://` Keycloak URL
- Both apps redirect to Keycloak and render the login form

→ [Step 19 — bootstrap and verify](19-bootstrap-and-verify.md)
