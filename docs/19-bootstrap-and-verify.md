# Step 19 — Bootstrap and verify

The cluster is running; it has no users and no energy community. This step
reaches parity with the docker-compose stack.

## 19.1 Create the Manager user

The admin portal needs a user holding the realm role `Manager`. Via the Keycloak
console (`https://keycloak.dev.yourdomain.com` → realm **EEGFaktura** → Users),
or scripted:

```bash
KC_ADMIN_PW=$(kubectl get secret eegfaktura-passwords -o jsonpath='{.data.keycloak-admin-password}' | base64 -d)
TOKEN=$(curl -s -X POST "$KC/realms/master/protocol/openid-connect/token" \
  -d client_id=admin-cli -d username=admin -d password="$KC_ADMIN_PW" \
  -d grant_type=password | jq -r .access_token)
```

> [!NOTE]
> **Mint the token immediately before you use it.** The five calls in 19.1 all
> carry `$TOKEN`, and a master-realm admin token lasts about a minute
> ([step 12.1](12-realm-config.md#121-get-an-admin-token)). A token left over
> from step 12, or one that aged while you read ahead, produces
>
> ```
> create user -> HTTP 401
> ```
>
> — not a permissions problem, and nothing is created. Re-run the two lines
> above, or the `kctoken` helper from 12.1, and repeat the call.

```bash
MANAGER_PW=$(LC_ALL=C tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 20); echo "manager password: $MANAGER_PW"
```

```bash
curl -s -o /dev/null -w "create user -> HTTP %{http_code}\n" \
  -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "$KC/admin/realms/EEGFaktura/users" -d "{
    \"username\":\"manager\",\"email\":\"manager@example.com\",
    \"enabled\":true,\"emailVerified\":true,
    \"credentials\":[{\"type\":\"password\",\"value\":\"$MANAGER_PW\",\"temporary\":false}]}"
```

```bash
USERID=$(curl -s -H "Authorization: Bearer $TOKEN" "$KC/admin/realms/EEGFaktura/users?username=manager" | jq -r '.[0].id')
ROLE=$(curl -s -H "Authorization: Bearer $TOKEN" "$KC/admin/realms/EEGFaktura/roles/Manager")
curl -s -o /dev/null -w "assign role -> HTTP %{http_code}\n" \
  -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "$KC/admin/realms/EEGFaktura/users/$USERID/role-mappings/realm" -d "[$ROLE]"
```

```bash
curl -s -H "Authorization: Bearer $TOKEN" "$KC/admin/realms/EEGFaktura/users/$USERID/role-mappings/realm" | jq -r '.[].name'
```

The list must include **Manager**. Save the generated password — it is not
printed again.

## 19.2 Register an EEG

Open `https://admin.dev.yourdomain.com`, log in as `manager`, and register one
with the sample values:

```
RC-Nummer:        TE100200
Gemeinschafts-ID: AT00999900000TC100200000000000002
Netzbetreiber-ID: AT009999
```

> [!TIP]
> Leave **EEG ist online** unchecked, and set **Ponton Kommunikation** to
> *ignorieren / bereits konfiguriert*. There is no Ponton gateway in this stack,
> and the master-data workflow does not need one.

Watch it happen:

```bash
kubectl logs -f deployment/eegfaktura-admin-backend | grep -vE "Validating token|Parsed token"
```

Success:

```
Create Keycloak User [exists=false} - TE100200
CREATE USER RESPONSE: 201
Keycloak User created true - TE100200
```

<details>
<summary>If registration returns HTTP 401</summary>

`NotAuthorizedException: HTTP 401`, with `invalid_client_credentials` in the
Keycloak log, means `KEYCLOAK_ADMIN_CLI_SECRET` is stale. The in-pod check in
[step 16.4](16-admin-backend.md#164-prove-the-credentials-work--before-you-need-them)
would have caught it; go back, regenerate, and restart:

```bash
kubectl rollout restart deployment/eegfaktura-admin-backend
```

</details>

## 19.3 Recover the generated password

Registration mints a platform user and e-mails the credentials — but the mail
relay is a placeholder, so read them from the log:

```bash
kubectl logs deployment/eegfaktura-admin-backend | grep "Create Keycloak-User"
```

```
... User(Some(<username>),<first>,<last>,<email>,Some(<password>))
```

## 19.4 Confirm the EEG reached the database

```bash
kubectl exec -it deployment/eegfaktura-postgresql -- psql -U eegfaktura -d eegfaktura \
  -c 'select tenant, name, "rcNumber", "communityId" from base.eeg;'
```

One row.

## 19.5 Import the master data

The two sample workbooks are in the compose repo you cloned in
[step 07](07-toolchains.md):

```bash
ls -la ~/src/eegfaktura-docker-compose/data/
```

```
TE100200-Muster-Stammdatenimport.xlsx          Stammdaten (participants)
TEST_EEG_Report_AT00999900000TE100100.xlsx     Energiedaten (meter readings)
```

They must be on the machine running the **browser**. If that is not the VM,
either copy them across or clone the compose repo locally:

```bash
scp <user>@"$VM_IP":'~/src/eegfaktura-docker-compose/data/*.xlsx' .
```

Open the app, log in with the credentials from 19.3, and upload **Stammdaten
first** — it creates the participants that the energy data attaches to — then
Energiedaten. Print the URL rather than reconstructing it by hand:

```bash
echo "https://$APP_HOST"
```

> [!IMPORTANT]
> **"Welcome to nginx!" means the `tlstest` scaffolding from
> [step 5.5](05-tls.md#55-prove-traefik-serves-the-certificate) is still
> deployed.** Nothing in this platform runs nginx — the web image is Caddy
> (`eegfaktura-web/Dockerfile`) — but 5.5's throwaway Ingress claims
> **`$APP_HOST` at path `/`**, exactly what `app-root` in `90-ingress.yaml`
> claims. Two Ingresses for the same host and path, and Traefik serves one of
> them; when `tlstest` wins you get nginx's default page instead of the app.
>
> ```bash
> kubectl get ingress,svc,deploy -n eegfaktura | grep -i tlstest
> ```
>
> Any output means 5.5's cleanup line was skipped or only partly ran:
>
> ```bash
> kubectl delete ingress tlstest -n eegfaktura
> kubectl delete service tlstest -n eegfaktura
> kubectl delete deployment tlstest -n eegfaktura
> ```
>
> If `tlstest` is already gone, then the request never reached this cluster —
> Traefik answers an unknown host with a 404, not that page. Check the name:
> `$BASE_DOMAIN` already carries the `dev.` label, so the app is at
> `app.dev.example.com`, not `app.example.com`.
>
> ```bash
> dig +short "$APP_HOST"    # must be $VM_IP
> ```

```bash
kubectl exec -it deployment/eegfaktura-postgresql -- psql -U eegfaktura -d eegfaktura \
  -c 'select count(*) as participants from base.participant;'
```

**A non-zero count is the finish line.** It means ingress → web → backend →
filestore → database all work, on images you built from source.

<details>
<summary>If the UI is empty, or the console shows <code>unexpected end of data at line 1 column 1 of the JSON data</code></summary>

Login succeeded and the API calls are being rejected. Check the backend:

```bash
kubectl logs deployment/eegfaktura-backend --tail=50 | grep -i unauthorized
```

`Request has no admin access group` means the `access_groups` claim is missing
from the token — the mapper from
[step 12.4](12-realm-config.md#124-add-the-access_groups-mapper) was not added,
or the user logged in before it was. **Log out and back in**: claims are fixed
when the token is issued.

Confirm what the token actually carries by decoding it in the browser console:

```js
JSON.parse(atob(JSON.parse(sessionStorage[Object.keys(sessionStorage).find(k=>k.startsWith('oidc.'))]).access_token.split('.')[1])).access_groups
```

It must list `/EEG_ADMIN`.

</details>

## 19.6 Parity checklist

| Check | Expect |
|---|---|
| `kubectl get pods` | all `Running`, restart counts not climbing |
| `https://app.dev.yourdomain.com` | loads, login works |
| `https://admin.dev.yourdomain.com` | loads, login works |
| `base.eeg` | one row |
| `base.participant` | > 0 after the import |
| Keycloak users | `manager` plus the generated EEG user |

> [!NOTE]
> **Metering points will stay in `INIT`/`NEW`, not `ACTIVE`.** That transition
> waits on an EDA response from a Ponton gateway, which this stack does not
> include — see [step 17.5](17-billing-eda.md#175-verify). Expected, not a
> fault in your cluster.

## Done when

The Excel import populates participants — the same end state as the
docker-compose stack, reached entirely with images you built yourself.

→ [Step 20 — the build pipeline](20-ci-pipeline.md)
