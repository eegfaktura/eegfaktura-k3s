# Step 12 — Realm configuration

The imported realm was exported from a different environment. Three things in it
do not match this deployment, and each one produces a failure that looks like
something else:

| Fix | Without it |
|---|---|
| Rotate the `admin-cli` secret | EEG registration fails with **HTTP 401** in [step 19](19-bootstrap-and-verify.md) |
| Add your redirect URIs | Login is rejected with **"Invalid parameter: redirect_uri"** |
| Add an `access_groups` mapper | Login *succeeds*, then every API call returns **401** and the UI stays empty |

All three are done through the Admin API, so they are scriptable and repeatable.

## 12.1 Get an admin token

```bash
KC_ADMIN_PW=$(kubectl get secret eegfaktura-passwords -o jsonpath='{.data.keycloak-admin-password}' | base64 -d)
TOKEN=$(curl -s -X POST "$KC/realms/master/protocol/openid-connect/token" \
  -d client_id=admin-cli -d username=admin -d password="$KC_ADMIN_PW" \
  -d grant_type=password | jq -r .access_token)
```

```bash
[ -n "$TOKEN" ] && [ "$TOKEN" != null ] && echo "token ok" || echo "TOKEN FAILED"
```

> [!NOTE]
> **Master-realm admin tokens are short-lived — a minute by default.** Expiry
> rarely announces itself as a plain `401`, because every command below pipes
> the response straight into `jq`. What you see instead is
>
> ```
> jq: error (at <stdin>:0): Cannot index object with number
> ```
>
> — `.[0]` applied to `{"error":"HTTP 401 Unauthorized"}` where an array of
> clients was expected. Re-mint the token and run the command again; whatever
> already returned `204` stayed applied. Since every section here needs it,
> keep it to one word:
>
> ```bash
> kctoken() { TOKEN=$(curl -s -X POST "$KC/realms/master/protocol/openid-connect/token" \
>   -d client_id=admin-cli -d username=admin -d password="$KC_ADMIN_PW" -d grant_type=password \
>   | jq -r .access_token); [ -n "$TOKEN" ] && [ "$TOKEN" != null ] && echo "token ok" || echo "TOKEN FAILED"; }
> ```

## 12.2 Rotate the `admin-cli` secret

The secret shipped in `keycloak.json` does not match the realm; using it fails
with `unauthorized_client`
([known problems #4](known-problems.md#4-the-shipped-admin-cli-secret-is-invalid)).
Regenerate it and capture the value — [step 16](16-admin-backend.md) needs it.

```bash
CID=$(curl -s -H "Authorization: Bearer $TOKEN" "$KC/admin/realms/EEGFaktura/clients?clientId=admin-cli" | jq -r '.[0].id')
ADMIN_CLI_SECRET=$(curl -s -X POST -H "Authorization: Bearer $TOKEN" "$KC/admin/realms/EEGFaktura/clients/$CID/client-secret" | jq -r .value)
echo "$ADMIN_CLI_SECRET"
```

> [!IMPORTANT]
> **Only the POST returns the plaintext.** A later `GET` on the client shows
> `**********`. Capture it now or regenerate again.

Store it where the admin backend will read it:

```bash
kubectl create secret generic eegfaktura-admin-cli --from-literal=secret="$ADMIN_CLI_SECRET"
```

```bash
kubectl get secret eegfaktura-admin-cli -o jsonpath='{.data.secret}' | base64 -d | head -c 8; echo "..."
```

## 12.3 Register your redirect URIs

The realm pins redirect URIs to the compose ports (`http://localhost:8001/*`,
`http://localhost:8002/*`), so authentication from your own hostnames is
rejected outright.

```bash
for pair in "at.ourproject.vfeeg.app:$APP_HOST" "at.ourproject.vfeeg.admin:$ADMIN_HOST"; do
  cid="${pair%%:*}"; host="${pair##*:}"
  ID=$(curl -s -H "Authorization: Bearer $TOKEN" "$KC/admin/realms/EEGFaktura/clients?clientId=$cid" | jq -r '.[0].id')
  curl -s -H "Authorization: Bearer $TOKEN" "$KC/admin/realms/EEGFaktura/clients/$ID" \
    | jq --arg u "https://$host" '.redirectUris = (.redirectUris + [$u + "/*", $u, $u + "/"] | unique)
                                | .webOrigins   = (.webOrigins   + [$u] | unique)' > "/tmp/client-$host.json"
  curl -s -o /dev/null -w "$cid -> HTTP %{http_code}\n" -X PUT \
    -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    "$KC/admin/realms/EEGFaktura/clients/$ID" -d @"/tmp/client-$host.json"
done
```

Both must report `HTTP 204`.

```bash
for cid in at.ourproject.vfeeg.app at.ourproject.vfeeg.admin; do curl -s -H "Authorization: Bearer $TOKEN" "$KC/admin/realms/EEGFaktura/clients?clientId=$cid" | jq -r '.[0] | .clientId, (.redirectUris|join(" ")), (.webOrigins|join(" "))'; done
```

The patch **appends**, so the existing `localhost` entries survive — nothing
that worked before breaks. Both the bare origin and the `/*` form are added
because the app's `redirect_uri` is `window.location.origin`, with no trailing
slash.

## 12.4 Add the `access_groups` mapper

> [!CAUTION]
> This is the subtlest defect in the whole stack, because it appears *after* a
> successful login. `eegfaktura-backend` authorises every protected route on a
> group named `/EEG_ADMIN`, read from an **`access_groups`** claim:
> ```go
> func (ag AccessGroups) IsAdmin() bool {
>     for _, s := range ag {
>         if s == "/EEG_ADMIN" { return true }
>     }
>     return false
> }
> ```
> `realm-export.json` contains **zero** occurrences of `access_groups`. The user
> genuinely holds the group in Keycloak; the claim simply never reaches the
> token. The symptom is `Unauthorized access with tenant … - Request has no
> admin access group` in the backend log, and in the browser an empty UI or
> `unexpected end of data at line 1 column 1 of the JSON data`. See
> [known problems #20](known-problems.md#20-the-realm-never-emits-the-claim-the-backend-requires).

```bash
APP_CID=$(curl -s -H "Authorization: Bearer $TOKEN" "$KC/admin/realms/EEGFaktura/clients?clientId=at.ourproject.vfeeg.app" | jq -r '.[0].id')
echo "$APP_CID"
```

```bash
curl -s -o /dev/null -w "mapper -> HTTP %{http_code}\n" -X POST \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "$KC/admin/realms/EEGFaktura/clients/$APP_CID/protocol-mappers/models" \
  -d '{"name":"access_groups","protocol":"openid-connect",
       "protocolMapper":"oidc-group-membership-mapper",
       "config":{"claim.name":"access_groups","full.path":"true",
                 "access.token.claim":"true","id.token.claim":"true",
                 "userinfo.token.claim":"true"}}'
```

`HTTP 201`. (`409` means it already exists — fine.)

`full.path` must be `true`: the backend compares against the leading-slash form
`/EEG_ADMIN`, not the bare group name.

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$KC/admin/realms/EEGFaktura/clients/$APP_CID/protocol-mappers/models" | jq -r '.[] | "\(.name)  \(.protocolMapper)"'
```

`access_groups  oidc-group-membership-mapper` must appear.

> [!NOTE]
> Claims are fixed when a token is issued. Anyone already logged in must log out
> and back in before the claim appears.

## 12.5 Confirm the realm's groups exist

The mapper only emits what the user actually has:

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$KC/admin/realms/EEGFaktura/groups" | jq -r '.[].path'
```

Expect `/EEG_ADMIN` and `/EEG_OWNER`. [Step 19](19-bootstrap-and-verify.md)
creates users into them.

## Done when

- `kubectl get secret eegfaktura-admin-cli` exists and holds the freshly rotated value
- Both clients list your `https://` origins in `redirectUris` and `webOrigins`
- `at.ourproject.vfeeg.app` has an `access_groups` group-membership mapper

→ [Step 13 — ingress and routing](13-ingress.md)
