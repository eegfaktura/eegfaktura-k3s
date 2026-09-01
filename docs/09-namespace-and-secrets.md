# Step 09 — Namespace, secrets and config

The compose stack uses file-based Docker secrets mounted at `/run/secrets/<name>`.
Kubernetes Secrets mounted at the **same paths** let the images work unchanged.

The namespace and the TLS secret already exist from steps 04 and 05. This step
adds the passwords and the two Keycloak config objects.

```bash
kubectl config view --minify -o jsonpath='{..namespace}'; echo
```

Must print `eegfaktura`. Everything below lands in the current namespace.

## 9.1 Regenerate the passwords

The compose repo ships password files, but two are obvious placeholders
(`YourSecretPassword` for SMTP, `SuperSecretPassword` for the Keycloak admin)
and all of them are committed to a **public repository**. Replace them.

> [!IMPORTANT]
> Do this **before** deploying Postgres. `initdb` runs once, on an empty data
> directory, and bakes in whatever password it is given. Changing it afterwards
> means deleting the PVC and starting over. Now is the cheap moment.

```bash
cd ~/src/eegfaktura-docker-compose
```

```bash
for f in eegfaktura-postgres-password eegfaktura-db-password \
         eegfaktura-keycloak-db-password eegfaktura-smtp-password \
         eegfaktura-keycloak-password; do
  printf '%s' "$(LC_ALL=C tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 24)" > "$f.txt"
done
```

Two details in that loop matter:

- **Alphanumeric only** — avoids breaking JDBC URLs and HOCON quoting downstream.
- **`printf '%s'`** writes no trailing newline. The `*_PASSWORD_FILE` variables
  read the file verbatim, so a stray `0a` becomes part of the password and
  authentication fails in a way that looks like a wrong password.

Verify that:

```bash
for f in eegfaktura-postgres-password eegfaktura-db-password eegfaktura-keycloak-db-password eegfaktura-smtp-password eegfaktura-keycloak-password; do printf "%-38s " "$f.txt"; xxd "$f.txt" | tail -1; done
```

No line may end in `0a`. The names are spelled out rather than globbed so that
`eegfaktura-keycloak-password` and `eegfaktura-keycloak-db-password` cannot be
confused — they are different credentials.

### Propagate the database password to the EDA config

`eegfaktura-eda.application.conf` carries the database password as a literal in
its Slick block, and becomes a ConfigMap in [step 17](17-billing-eda.md). It must
be updated or EDA cannot reach the database.

```bash
NEW_DB_PW=$(cat eegfaktura-db-password.txt)
sed -i "s/Dzy5lShLn1N3rqTM/$NEW_DB_PW/g" eegfaktura-eda.application.conf
```

```bash
grep -n 'password=' eegfaktura-eda.application.conf
```

The committed literal must be gone. The `sed` is idempotent — running it twice
is harmless.

> [!NOTE]
> **`docker-compose.yaml` needs no edit.** Its five password literals are
> reference only. The manifests use `secretKeyRef`, so the Secret is the single
> source of truth.

## 9.2 Create the password Secret

```bash
kubectl delete secret eegfaktura-passwords --ignore-not-found
```

```bash
kubectl create secret generic eegfaktura-passwords \
  --from-file=eegfaktura-postgres-password=eegfaktura-postgres-password.txt \
  --from-file=eegfaktura-db-password=eegfaktura-db-password.txt \
  --from-file=eegfaktura-keycloak-db-password=eegfaktura-keycloak-db-password.txt \
  --from-file=eegfaktura-smtp-password=eegfaktura-smtp-password.txt \
  --from-file=keycloak-admin-password=eegfaktura-keycloak-password.txt
```

```bash
kubectl get secret eegfaktura-passwords -o jsonpath='{.data}' | jq 'keys'
```

Five keys. `keycloak-admin-password` has no equivalent in compose — there the
Keycloak admin password is hardcoded and `eegfaktura-keycloak-password.txt` goes
unused. Wiring it in removes the last credential literal from the manifests.

> [!CAUTION]
> **Do not commit the regenerated files.** They live in a clone of a public
> repository. Keep the changes local:
> ```bash
> git -C ~/src/eegfaktura-docker-compose status --short
> ```

## 9.3 The Keycloak client config

> [!WARNING]
> **This file hardcodes the compose hostname.** All four sections (`app`, `api`,
> `admin`, `admin-cli`) carry
> `"auth-server-url": "http://eegfaktura-keycloak:8080"`. The backend and
> energystore read it via `KEYCLOAK_CONFIG` to fetch JWKS — from a host that does
> not exist in the cluster. The backend hangs, then panics:
> ```
> panic: Get "http://eegfaktura-keycloak:8080/realms/EEGFaktura/.well-known/openid-configuration":
> context deadline exceeded
> ```
> This is a **separate file** from `realm-export.json`. Both need rewriting, and
> fixing only one leaves the other to fail later. See
> [known problems #6](known-problems.md#6-two-config-files-hardcode-the-compose-hostname).

```bash
sed -i "s|http://eegfaktura-keycloak:8080|$KC|g" keycloak/keycloak.json
```

```bash
grep -o '"auth-server-url"[^,]*' keycloak/keycloak.json
```

Every occurrence must now read `https://keycloak.<your-domain>`.

```bash
kubectl create secret generic eegfaktura-keycloak-json \
  --from-file=keycloak.json=keycloak/keycloak.json
```

> [!NOTE]
> **The admin backend ignores this file.** `eeg-registration-backend` has
> `configfile = ${?KEYCLOAK_CONFIG_JSON}` commented out in its bundled
> `application.conf`, so mounting it there achieves nothing. Its secret comes
> from the `KEYCLOAK_ADMIN_CLI_SECRET` environment variable instead — see
> [step 12](12-realm-config.md) and
> [known problems #5](known-problems.md#5-eeg-registration-backend-ignores-its-mounted-config).

## 9.4 The realm import

Compose bind-mounts `./keycloak/import`. Here it becomes a ConfigMap — the
export is ~79 KB, comfortably under the 1 MB limit.

> [!WARNING]
> **The export hardcodes the compose hostname too — fix it before importing.**
> `realm-export.json` sets a realm-level `frontendUrl`:
> ```json
> "attributes": { "frontendUrl": "http://eegfaktura-keycloak:8080" }
> ```
> This **overrides** the server-level `KC_HOSTNAME`, so Keycloak advertises the
> compose hostname in every token's `iss` and in
> `.well-known/openid-configuration` no matter what the Deployment sets. Token
> validation then fails everywhere, and re-importing does not fix it — the realm
> import strategy is `IGNORE_EXISTING`.

```bash
sed -i "s|http://eegfaktura-keycloak:8080|$KC|g" keycloak/import/realm-export.json
```

```bash
grep -o '"frontendUrl"[^,]*' keycloak/import/realm-export.json
```

```bash
kubectl create configmap keycloak-realm-import \
  --from-file=realm-export.json=keycloak/import/realm-export.json
```

## 9.5 Verify

```bash
kubectl get secret,configmap -n eegfaktura
```

Expect `eegfaktura-tls`, `eegfaktura-passwords`, `eegfaktura-keycloak-json` and
the `keycloak-realm-import` ConfigMap (plus the automatic `kube-root-ca.crt`).

```bash
cd ~
```

## Done when

- Three Secrets and one ConfigMap exist **in namespace `eegfaktura`**
- No password file ends in a newline
- Neither Keycloak config file mentions `eegfaktura-keycloak:8080`

→ [Step 10 — PostgreSQL](10-postgres.md)
