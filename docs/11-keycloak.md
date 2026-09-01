# Step 11 — Keycloak

**The step everything else hangs on.** Keycloak stamps its own URL into the
`iss` claim of every token, and the backends fetch signing keys (JWKS) from that
same URL. If the browser and the pods disagree about Keycloak's hostname, every
login fails with errors that point everywhere except the real cause.

One name — `keycloak.dev.yourdomain.com` — reachable from both sides.

Manifest: **`k8s/20-keycloak.yaml`** — Deployment, Service and Ingress.

## 11.1 What the manifest does and why

**`KC_HOSTNAME: https://keycloak.<your-domain>`** — the string that ends up in
`iss`. Everything downstream is compared against it.

**The Service listens on port 80, not 8080.** Traefik terminates TLS and
forwards plain HTTP to the pod, so 80 is the port that answers behind the
`https://` URL. A pod requesting `https://keycloak.<your-domain>/…` goes back
out through Traefik, which is where the port mapping happens.

**`KC_PROXY_HEADERS: xforwarded`** — without it Keycloak builds redirect URLs
from the internal request rather than the original `https://` one, and login
bounces back to `http://`.

**`KC_BOOTSTRAP_ADMIN_USERNAME` / `_PASSWORD`** — Keycloak 26 renamed the old
`KEYCLOAK_ADMIN` pair. The password comes from the Secret created in step 09,
so no credential is written in the manifest.

## 11.2 Deploy

```bash
kubectl apply -f "$K8S"/20-keycloak.yaml
```

```bash
kubectl rollout status deployment/eegfaktura-keycloak --timeout=300s
```

First boot imports the realm and runs Keycloak's own schema migrations against
the `keycloak` database. Several minutes is normal:

```bash
kubectl logs -f deployment/eegfaktura-keycloak
```

<details>
<summary>Startup warnings you can ignore</summary>

```
KC-SERVICES0110: Environment variable 'KEYCLOAK_ADMIN' is deprecated,
                 use 'KC_BOOTSTRAP_ADMIN_USERNAME' instead
```
Comes from the image's own defaults; the manifest already uses the new names.
The bootstrap admin is created only on first boot either way.

```
WARNING: Hostname v1 options [hostname-path, ..., proxy, ...] are still in use
```
Baked into the image's build-time options, not set by this manifest. Cosmetic.

</details>

## 11.3 Verify the issuer from outside

```bash
curl -s "$KC/realms/EEGFaktura/.well-known/openid-configuration" | jq -r '.issuer, .jwks_uri'
```

Both must start with `https://keycloak.<your-domain>`. No `-k` flag — the
certificate from [step 05](05-tls.md) must validate on its own.

<details>
<summary>If the issuer shows <code>eegfaktura-keycloak:8080</code></summary>

The realm's own `frontendUrl` attribute is overriding `KC_HOSTNAME` — the
`sed` in [step 9.4](09-namespace-and-secrets.md#94-the-realm-import) did not take
effect before the realm was imported. Re-importing will **not** fix it: the
startup log shows `Strategy: IGNORE_EXISTING`, so an existing realm is left
alone. Patch the live realm:

```bash
KC_ADMIN_PW=$(kubectl get secret eegfaktura-passwords -o jsonpath='{.data.keycloak-admin-password}' | base64 -d)
TOKEN=$(curl -s -X POST "$KC/realms/master/protocol/openid-connect/token" \
  -d client_id=admin-cli -d username=admin -d password="$KC_ADMIN_PW" \
  -d grant_type=password | jq -r .access_token)

curl -s -H "Authorization: Bearer $TOKEN" "$KC/admin/realms/EEGFaktura" \
  | jq --arg u "$KC" '.attributes.frontendUrl = $u' > /tmp/realm.json

curl -s -o /dev/null -w "realm -> HTTP %{http_code}\n" -X PUT \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  "$KC/admin/realms/EEGFaktura" -d @/tmp/realm.json
```

GET-modify-PUT preserves the other realm attributes. Fix the ConfigMap source
too, so a rebuilt cluster is correct from the start.

</details>

## 11.4 Verify the issuer from inside a pod

This is the half that the compose stack never has to think about, and the half
that breaks silently.

```bash
kubectl run curltest --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s "https://$KC_HOST/realms/EEGFaktura/.well-known/openid-configuration" \
  | grep -o '"issuer":"[^"]*"'
```

It must print **exactly** the same issuer as 11.3.

> [!NOTE]
> The two sides may resolve to different addresses — that is fine. Only the
> *string* appears in `iss`, so token validation matches either way. What must
> never differ is the hostname.

<details>
<summary>If the in-pod call fails</summary>

First find out which half is broken:

```bash
kubectl run dnstest --rm -it --restart=Never --image=busybox:1.36 -- nslookup keycloak.dev.yourdomain.com
```

**`NXDOMAIN`** — pods cannot resolve the name. CoreDNS forwards to the VM's
resolver, which does not know it. This is what happens if you used the
`/etc/hosts` fallback in [step 02](02-domain-and-dns.md), since CoreDNS never
reads those entries. Give CoreDNS the mapping directly:

```bash
kubectl apply -f "$K8S"/05-coredns-custom.yaml
kubectl -n kube-system rollout restart deployment/coredns
kubectl -n kube-system rollout status deployment/coredns
```

```bash
kubectl run dnstest --rm -it --restart=Never --image=busybox:1.36 -- nslookup keycloak.dev.yourdomain.com
```

It must now return `$VM_IP`. Then re-run the curl above.

<br>

**An address, but the curl still fails** — DNS is fine and the problem is the
hairpin: pod → node IP → back into the cluster. Some CNI and firewall
combinations drop that. Test it plainly:

```bash
kubectl run curltest --rm -it --restart=Never --image=curlimages/curl -- \
  curl -s -o /dev/null -w '%{http_code}\n' "https://$KC_HOST/realms/EEGFaktura"
```

If this cannot be made to work, the alternative is to terminate TLS in Keycloak
itself and point a CoreDNS `rewrite` at the Service on 443 — more moving parts,
so try the hairpin first.

<br>

**A TLS error** — the certificate is not trusted inside the pod. Confirm you
issued a publicly-trusted certificate in step 05 and not a self-signed one; the
Go and Java clients in the backends will reject it too.

</details>

> [!IMPORTANT]
> Do not move on until both checks return the same `https://` issuer. Every
> authentication failure in steps 15–19 traces back to this.

## 11.5 Log in to the admin console

```bash
kubectl get secret eegfaktura-passwords -o jsonpath='{.data.keycloak-admin-password}' | base64 -d; echo
```

Browse to `https://keycloak.dev.yourdomain.com` and log in as `admin`.

Confirm the **EEGFaktura** realm exists, with clients
`at.ourproject.vfeeg.app`, `at.ourproject.vfeeg.admin` and `admin-cli`.

## Done when

- Both issuer checks — outside and in-pod — return the identical `https://` URL
- The admin console loads in your browser with a valid certificate
- The `EEGFaktura` realm and its three clients are present

→ [Step 12 — realm configuration](12-realm-config.md)
