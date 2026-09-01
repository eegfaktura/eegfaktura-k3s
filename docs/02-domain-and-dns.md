# Step 02 — Domain and DNS

Three hostnames point at the VM. Getting them resolvable — for your browser
**and** for the pods inside the cluster — is the foundation everything else
stands on, so it comes before the cluster itself.

| Hostname | Serves | Replaces (compose) |
|---|---|---|
| `keycloak.dev.yourdomain.com` | Keycloak | `eegfaktura-keycloak:8080` |
| `app.dev.yourdomain.com` | platform UI | `localhost:8001` |
| `admin.dev.yourdomain.com` | admin portal | `localhost:8002` |

## 2.1 Why real hostnames, and why one of them matters more

Keycloak stamps its own URL into the `iss` claim of every token it issues, and
every backend fetches signing keys (JWKS) from that same URL. If the browser and
the pods disagree about what Keycloak is called, token validation fails — with
errors that point everywhere except the real cause.

The rule that follows: **one hostname, resolvable from both sides.** The
*addresses* may differ; the *string* may not, because the string is what ends up
in `iss`.

<details>
<summary>Why not a wildcard DNS service like <code>nip.io</code> or <code>sslip.io</code></summary>

They derive the address from the hostname and would avoid all DNS setup, but
they usually fail on a private network. Measured against a consumer router
acting as resolver:

```console
$ dig +short nip.io              # 78.46.204.247   the service itself resolves
$ dig +short 8.8.8.8.nip.io      # 8.8.8.8         public addresses come back
$ dig +short 192.168.1.50.nip.io # (empty)         private addresses are stripped
```

That is **DNS rebinding protection**: the resolver discards upstream answers
pointing into private ranges. Since the VM has a private address, every such
name comes back empty.

Using a domain you control sidesteps it — and is required anyway, because
[step 05](05-tls.md) needs to prove control of the domain to get a certificate.

</details>

## 2.2 The same variables on your workstation

A few commands below and in later steps are run from the machine with the
browser, not the VM. Export the same two values there so those can be pasted
unchanged too — in `~/.zshrc` on a current macOS, `~/.bashrc` on Linux:

```bash
export VM_IP=192.168.1.50
export BASE_DOMAIN=dev.yourdomain.com
export KC_HOST="keycloak.$BASE_DOMAIN"
export APP_HOST="app.$BASE_DOMAIN"
export ADMIN_HOST="admin.$BASE_DOMAIN"
```

Use the identical values you set on the VM in
[step 1.3](01-vm.md#13-record-your-settings). This guide marks which side each
command runs on.

## 2.3 Make the names resolve on your LAN

Pick whichever matches your network. All three produce the same result: the
three names resolve to `$VM_IP` for every device on the LAN, **including the
VM itself**, which is what lets pods resolve them too.

<details open>
<summary><b>Option A — your router or local resolver (recommended)</b></summary>

One entry, and every device on the LAN picks it up. No per-machine files.

**OPNsense / pfSense (Unbound)** — *Services → Unbound DNS → Overrides → Host
Overrides → +*, one row per name:

| Host | Domain | Type | IP |
|---|---|---|---|
| `keycloak` | `dev.yourdomain.com` | A | `192.168.1.50` |
| `app` | `dev.yourdomain.com` | A | `192.168.1.50` |
| `admin` | `dev.yourdomain.com` | A | `192.168.1.50` |

Or cover the whole zone at once — *Services → Unbound DNS → General → Custom
options*:

```
server:
  local-zone: "dev.yourdomain.com." redirect
  local-data: "dev.yourdomain.com. 3600 IN A 192.168.1.50"
```

`redirect` answers **any** name under the zone with that record, so future
services need no new entries.

**dnsmasq / Pi-hole / OpenWrt** — add to `dnsmasq.conf`:

```
address=/dev.yourdomain.com/192.168.1.50
```

**Why this survives rebinding protection.** A resolver serves its own overrides
as authoritative local data, so it does not filter them. A *public* A record
pointing at a private address would still be stripped — exactly the behaviour
measured above. Keep these records local.

</details>

<details>
<summary><b>Option B — public DNS records</b></summary>

If the VM has a public address, or your resolver does not filter private
answers, publish three A records at your DNS provider pointing at `$VM_IP`.
Simplest to reason about, and pods get it for free.

Check first that the answers are not being stripped:

```bash
dig +short app.$BASE_DOMAIN
```

An empty answer for a private address means your resolver filters them; use
option A instead.

</details>

<details>
<summary><b>Option C — <code>/etc/hosts</code> (fallback)</b></summary>

Works, but must be repeated on every machine that browses the apps — **and pods
never see it**, so [step 11](11-keycloak.md) will need the CoreDNS override.

Run this on your workstation **and** on the VM — the variables from 2.2 and 1.3
must be set in the shell you run it from, because the heredoc expands them:

```bash
sudo tee -a /etc/hosts <<EOF
$VM_IP  $KC_HOST
$VM_IP  $APP_HOST
$VM_IP  $ADMIN_HOST
EOF
```

```bash
tail -3 /etc/hosts
```

Confirm the three lines carry a real address and real names — a line reading
just a bare hostname means the variables were not set.

</details>

## 2.4 Verify — from both the VM and your workstation

On the **VM**:

```bash
for h in "$KC_HOST" "$APP_HOST" "$ADMIN_HOST"; do printf '%-40s %s\n' "$h" "$(getent hosts "$h" | awk '{print $1}')"; done
```

On your **workstation**:

```bash
for h in "$KC_HOST" "$APP_HOST" "$ADMIN_HOST"; do printf '%-40s %s\n' "$h" "$(dig +short "$h")"; done
```

All six must return the VM's address.

> [!IMPORTANT]
> The VM resolving these names is not optional garnish — CoreDNS inside the
> cluster forwards to the VM's resolver, so this is what will let pods reach
> Keycloak in [step 11](11-keycloak.md).

## 2.5 A DNS API token for the certificate

[Step 05](05-tls.md) proves domain control by writing a temporary `_acme-challenge`
TXT record, so it needs API access to wherever `$BASE_DOMAIN` is hosted. Nothing
inbound is required, which is why this works for a VM on a private address.

`lego` supports well over a hundred providers — Cloudflare, deSEC, Route 53,
Hetzner, DigitalOcean, Gandi, and so on. Create a scoped API token at your
provider now, and check the variable names it expects:

```bash
lego dnshelp -c <provider>
```

(`lego` gets installed in step 05; the examples there use **deSEC**, whose token
page is at <https://desec.io/tokens>.)

> [!NOTE]
> If your `$BASE_DOMAIN` is a private-only name with no registrar behind it, you
> cannot get a publicly-trusted certificate. Read the opening of
> [step 05](05-tls.md) before going further — a self-signed certificate is not
> a workable substitute here.

## Done when

- All three names resolve to `$VM_IP` from your workstation
- All three names resolve to `$VM_IP` **on the VM**
- You hold an API token for the DNS zone

→ [Step 03 — install k3s](03-k3s.md)
