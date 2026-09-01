# Step 05 — TLS

**This step is required, not a hardening pass to do later.**

The platform frontend cannot log in over plain HTTP. `oidc-client-ts` computes
an S256 PKCE challenge with `crypto.subtle`, and browsers expose Web Crypto only
in a **secure context**. `http://localhost:8001` qualifies — browsers trust
localhost unconditionally, which is why the docker-compose stack never hits this
— but `http://app.dev.yourdomain.com` does not:

```js
window.crypto.subtle   // undefined
```

`signinRedirect()` then throws before it can build the authorize URL: no
navigation, no network request, and no error message, because the app renders
its loading branch before its error branch. The page just spins. Full analysis
in [known problems #19](known-problems.md#19-the-frontend-requires-https-and-hides-the-failure).

Doing TLS now also means Keycloak is deployed with an `https://` issuer from the
start. Retrofitting it later means re-patching the realm, both frontends, the
admin backend, and the Keycloak client config — all of which must move together,
because a `https://` page cannot call an `http://` API without the browser
blocking it.

> [!CAUTION]
> **A self-signed certificate is not enough.** Pods fetch JWKS from
> `keycloak.dev.yourdomain.com` too. A browser lets you click through an
> untrusted certificate; the Go, Java and Scala HTTP clients in the backends do
> not. Use a publicly-trusted certificate so both sides work without custom
> trust stores.

## 5.1 Install lego

`lego` is a single-binary ACME client with DNS-01 support for a hundred-plus
providers. DNS-01 never needs an inbound connection, which is what makes a
publicly-trusted certificate possible for a VM on a private address.

Use the release binary, not the distro package — Ubuntu 24.04 ships lego
**4.9.1** (2022) against a current **5.x**, several majors behind.

The release assets embed the version in their filename, so the
`releases/latest/download/` shortcut does not work. Resolve the URL from the API:

```bash
LEGO_URL=$(curl -s https://api.github.com/repos/go-acme/lego/releases/latest | grep -o 'https://[^"]*linux_amd64\.tar\.gz' | head -1)
echo "$LEGO_URL"
```

```bash
curl -sL "$LEGO_URL" | tar xz lego
sudo mv lego /usr/local/bin/
lego --version
```

## 5.2 Issue a wildcard certificate

Find the environment variables your DNS provider needs:

```bash
lego dnshelp -c desec        # or cloudflare, route53, hetzner, gandiv5, ...
```

Export the credentials, then request the certificate. A wildcard covers all
three hostnames and anything you add later:

```bash
export DESEC_TOKEN='<your-api-token>'
export ACME_EMAIL='you@example.com'
```

```bash
lego --path ~/lego \
  --email "$ACME_EMAIL" --accept-tos \
  --dns desec \
  --domains "$BASE_DOMAIN" --domains "*.$BASE_DOMAIN" \
  run
```

> [!WARNING]
> **`--path` matters.** Without it lego writes to `./.lego`, relative to
> whatever directory you happened to be in — not `$HOME/.lego`. Renewals must
> use the same `--path`, so fixing it now avoids hunting for the certificate
> later.

```bash
ls ~/lego/certificates/
```

Expect `$BASE_DOMAIN.crt`, `.key`, `.issuer.crt` and `.json`.

<details>
<summary>If issuance fails</summary>

| Message | Cause |
|---|---|
| `unable to find a solver` | Wrong `--dns` provider name. Check `lego dnshelp` |
| `some credentials information are missing` | The env var names differ from what you exported — `lego dnshelp -c <provider>` lists the exact names |
| `timeout waiting for record` | Slow zone propagation. Retry with `--dns-timeout 120` |
| `too many certificates already issued` | Let's Encrypt rate limit — five per domain per week. Add `--server https://acme-staging-v02.api.letsencrypt.org/directory` while experimenting, but note the staging CA is **not trusted**, so re-issue against production before continuing |

</details>

Check what you got:

```bash
openssl x509 -in ~/lego/certificates/"$BASE_DOMAIN".crt -noout -subject -issuer -dates -ext subjectAltName
```

The SAN list must contain both `$BASE_DOMAIN` and `*.$BASE_DOMAIN`.

## 5.3 Load it into the cluster

```bash
kubectl create secret tls eegfaktura-tls -n eegfaktura \
  --cert ~/lego/certificates/"$BASE_DOMAIN".crt \
  --key  ~/lego/certificates/"$BASE_DOMAIN".key
```

```bash
kubectl get secret eegfaktura-tls -n eegfaktura
```

The `.crt` lego writes already contains the full chain, which is what Traefik
needs — no concatenation step.

> [!NOTE]
> **Renewal.** Let's Encrypt certificates last 90 days. Re-run the same command
> with `renew` instead of `run` and the same `--path`, then delete and recreate
> the Secret. For a permanent setup, install cert-manager with a DNS-01 solver
> and stop thinking about it; for a dev cluster a calendar reminder is adequate.

## 5.4 Redirect HTTP to HTTPS

So that typing `http://` never produces the silent spinner described above:

```bash
kubectl apply -f "$MANIFESTS"/01-traefik-redirect.yaml
```

```bash
kubectl -n kube-system rollout status deployment/traefik --timeout=180s
```

k3s reinstalls Traefik from its bundled Helm chart with these values merged in.
It takes a few seconds and the ingress is briefly unavailable.

<details>
<summary>If your k3s does not accept <code>HelmChartConfig</code></summary>

Older k3s releases, or a cluster installed with `--disable traefik` and a
hand-managed Traefik, will not pick this up. It is a convenience, not a
requirement — skip it and always type `https://`. Everything else in this guide
works either way.

</details>

## 5.5 Prove Traefik serves the certificate

Worth two minutes now, rather than discovering a bad certificate at
[step 18](18-frontends.md). This reuses the smoke test from step 03, with TLS:

```bash
kubectl create deployment tlstest --image=nginx -n eegfaktura
kubectl expose deployment tlstest --port=80 -n eegfaktura
```

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tlstest
  namespace: eegfaktura
spec:
  ingressClassName: traefik
  tls:
    - hosts: ["$APP_HOST"]
      secretName: eegfaktura-tls
  rules:
    - host: $APP_HOST
      http:
        paths:
          - {path: /, pathType: Prefix, backend: {service: {name: tlstest, port: {number: 80}}}}
EOF
```

```bash
kubectl rollout status deployment/tlstest -n eegfaktura --timeout=120s
```

```bash
curl -sI "https://$APP_HOST" | head -1
```

```bash
curl -sI "http://$APP_HOST" | head -3
```

The first must be `200`; the second `301` with a `Location: https://…` — proving
both the certificate and the redirect. **No `-k` flag**: the point is that the
certificate validates on its own.

Then open `https://app.dev.yourdomain.com` in your browser — a padlock, no
warning — and in the console:

```js
window.crypto.subtle    // an object, not undefined
```

That is the condition [step 18](18-frontends.md) depends on.

Clean up:

```bash
kubectl delete ingress tlstest -n eegfaktura; kubectl delete service tlstest -n eegfaktura; kubectl delete deployment tlstest -n eegfaktura
```

## Done when

- `kubectl get secret eegfaktura-tls -n eegfaktura` exists
- `curl -sI https://app.dev.yourdomain.com` returns 200 without `-k`
- `window.crypto.subtle` is defined in the browser

→ [Step 06 — the registry](06-registry.md)
