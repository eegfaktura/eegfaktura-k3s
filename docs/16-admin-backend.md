# Step 16 — Admin backend (Scala)

This service registers EEGs and creates their Keycloak users. It is where the
`admin-cli` secret rotated in [step 12](12-realm-config.md) is finally used.

Manifest: **`k8s/50-admin-backend.yaml`**

## 16.1 Build with sbt

There is **no Dockerfile**. The image is produced by sbt-native-packager
(`JavaAppPackaging`, base `eclipse-temurin:17-jre`):

```bash
cd ~/src/eegfaktura-admin-backend
sbt Docker/publishLocal
```

The first run downloads the Scala toolchain and dependencies — expect 10+
minutes and high memory use.

<details>
<summary>If sbt runs out of heap</summary>

```bash
export SBT_OPTS="-Xmx4G -XX:+UseG1GC"
sbt Docker/publishLocal
```

</details>

sbt names the image for upstream's registry, so retag it into yours.
`dockerVersion` in `build.sbt` sets the tag — capture it rather than typing it:

```bash
docker images | grep -i registration
```

```bash
REG_TAG=$(docker images --format '{{.Tag}}' ghcr.io/vfeeg-development/eeg-registration-backend | grep -v latest | head -1)
echo "$REG_TAG"
```

```bash
docker tag ghcr.io/vfeeg-development/eeg-registration-backend:"$REG_TAG" localhost:5000/eeg-registration-backend:dev
docker push localhost:5000/eeg-registration-backend:dev
```

> [!TIP]
> **Never paste a `<version>` placeholder into a shell.** Bash reads `<` as an
> input redirect, so `docker tag …:<version>` fails with a confusing syntax
> error rather than "you forgot to substitute something". Capturing the value
> into a variable sidesteps it.

```bash
cd ~
```

## 16.2 The setting that actually matters

> [!CAUTION]
> **Mounting `keycloak.json` here does nothing.** The bundled `application.conf`
> has the line that would load it commented out:
> ```hocon
> #  configfile = ${?KEYCLOAK_CONFIG_JSON}
> secret = "P85u55EUB7w6JFjQBxsHDCbdy8TXibDI"   # hardcoded fallback
> secret = ${?KEYCLOAK_ADMIN_CLI_SECRET}        # the only working override
> ```
> Without `KEYCLOAK_ADMIN_CLI_SECRET`, the service authenticates with the stale
> secret baked into the JAR and EEG registration fails with
> `Creating Keycloak User failed! … NotAuthorizedException: HTTP 401`, and
> `error="invalid_client_credentials"` in the Keycloak log.
> [Known problems #5](known-problems.md#5-eeg-registration-backend-ignores-its-mounted-config).

The manifest reads it from the `eegfaktura-admin-cli` Secret created in
[step 12.2](12-realm-config.md#122-rotate-the-admin-cli-secret).

> [!NOTE]
> **`KEYCLOAK_URL` must be the public hostname**, not the internal Service name.
> It is used for both JWKS retrieval and issuer validation, so it has to match
> the `iss` in tokens minted for the browser.

## 16.3 Apply

```bash
kubectl apply -f "$K8S"/50-admin-backend.yaml
```

```bash
kubectl rollout status deployment/eegfaktura-admin-backend --timeout=300s
```

```bash
kubectl logs deployment/eegfaktura-admin-backend --tail=20
```

Look for `Server online at http://…:8085/`.

## 16.4 Prove the credentials work — before you need them

This check predicts whether EEG registration in
[step 19](19-bootstrap-and-verify.md) will succeed, and it is much easier to
read here than through the UI:

```bash
kubectl exec deployment/eegfaktura-admin-backend -- sh -c \
  'wget -qO- --post-data="client_id=admin-cli&client_secret=$KEYCLOAK_ADMIN_CLI_SECRET&grant_type=client_credentials" \
   --header="Content-Type: application/x-www-form-urlencoded" \
   "$KEYCLOAK_URL/realms/EEGFaktura/protocol/openid-connect/token"'
```

An `access_token` in the response means registration will work.

<details>
<summary>If it returns <code>invalid_client_credentials</code></summary>

The Secret does not match the realm. Regenerate and replace it:

```bash
KC_ADMIN_PW=$(kubectl get secret eegfaktura-passwords -o jsonpath='{.data.keycloak-admin-password}' | base64 -d)
TOKEN=$(curl -s -X POST "$KC/realms/master/protocol/openid-connect/token" -d client_id=admin-cli -d username=admin -d password="$KC_ADMIN_PW" -d grant_type=password | jq -r .access_token)
CID=$(curl -s -H "Authorization: Bearer $TOKEN" "$KC/admin/realms/EEGFaktura/clients?clientId=admin-cli" | jq -r '.[0].id')
ADMIN_CLI_SECRET=$(curl -s -X POST -H "Authorization: Bearer $TOKEN" "$KC/admin/realms/EEGFaktura/clients/$CID/client-secret" | jq -r .value)

kubectl delete secret eegfaktura-admin-cli
kubectl create secret generic eegfaktura-admin-cli --from-literal=secret="$ADMIN_CLI_SECRET"
kubectl rollout restart deployment/eegfaktura-admin-backend
```

</details>

<details>
<summary>If it fails on TLS or the connection</summary>

The pod cannot reach `$KEYCLOAK_URL`. That is the in-pod resolution problem from
[step 11.4](11-keycloak.md#114-verify-the-issuer-from-inside-a-pod), not a
credential problem — fix it there first.

</details>

## Done when

- Pod logs show `Server online at http://…:8085/`
- The in-pod token request returns an `access_token`

→ [Step 17 — billing and EDA](17-billing-eda.md)
