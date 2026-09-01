# Step 04 — The repo and your settings

The manifests are real files in this repository, not snippets to copy out of
markdown. Clone it onto the VM once; every later step applies files from
`$K8S`.

## 4.1 Clone

```bash
cd ~
git clone https://github.com/<owner>/eegfaktura-k3s.git
cd ~/eegfaktura-k3s
ls k8s/
```

Use your own fork if you intend to keep changes (see [4.5](#45-keeping-your-changes)),
or clone this repository directly to follow along.

`$K8S` was set to `~/eegfaktura-k3s/k8s` in [step 01](01-vm.md#13-record-your-settings).
Confirm it points at what you just cloned:

```bash
ls "$K8S"/*.yaml | wc -l      # 15
```

## 4.2 What is in here

```
docs/       this guide, one file per step, plus known-problems.md
k8s/        Kubernetes manifests, numbered like the upstream platform repo
```

The manifest numbers are independent of the step numbers — they follow the
convention the upstream deployment repo uses, where each service owns one
numbered file.

| Manifest | Applied in |
|---|---|
| `01-traefik-redirect.yaml` | 05 |
| `05-coredns-custom.yaml` | 11 — *only if* pods cannot resolve the Keycloak name |
| `10-postgres.yaml` | 10 |
| `20-keycloak.yaml` | 11 |
| `30-mosquitto.yaml`, `31-postfix.yaml`, `32-filestore.yaml` | 14 |
| `40-backend.yaml`, `41-energystore.yaml` | 15 |
| `50-admin-backend.yaml` | 16 |
| `60-billing.yaml`, `61-eda.yaml` | 17 |
| `70-web.yaml`, `71-admin-web.yaml` | 18 |
| `90-ingress.yaml` | 13 |

## 4.3 Substitute your domain and address

The manifests ship with the placeholders `dev.yourdomain.com` and
`192.168.1.50`. Replace both in one pass:

```bash
cd ~/eegfaktura-k3s
grep -rl -e 'dev\.yourdomain\.com' -e '192\.168\.1\.50' k8s/ | xargs -r sed -i "s/dev\.yourdomain\.com/$BASE_DOMAIN/g; s/192\.168\.1\.50/$VM_IP/g"
```

Verify nothing was missed, and that the result is what you expect:

```bash
grep -rn "yourdomain\.com\|192\.168\.1\.50" k8s/ | grep -v example.com
```

```bash
grep -rho "[a-z]*\.$BASE_DOMAIN" k8s/ | sort -u
```

The first command must print **nothing**. The second must print exactly your
three hostnames.

> [!NOTE]
> `smtp.example.com` and `relay-user@example.com` in `31-postfix.yaml` are
> deliberately left alone — they are the upstream mail-relay placeholders, and
> nothing in this guide needs working outbound mail. See
> [step 14](14-support-services.md).

## 4.4 Namespace

Create it now and make it the default for your session, so no later command
needs `-n eegfaktura`:

```bash
kubectl create namespace eegfaktura
kubectl config set-context --current --namespace=eegfaktura
kubectl config view --minify -o jsonpath='{..namespace}'; echo
```

The last command must print `eegfaktura`. If it errors with
`k3s.yaml.lock: permission denied`, go back to
[step 3.2](03-k3s.md#32-give-yourself-a-writable-kubeconfig) — the kubeconfig
copy was skipped.

> [!IMPORTANT]
> Verify the namespace actually switched. If it did not, everything from
> [step 09](09-namespace-and-secrets.md) onward lands in `default`, and services
> fail to find each other in ways that look like application bugs.

## 4.5 Keeping your changes

The substituted manifests are now local modifications:

```bash
git status --short
```

They are yours to keep. If you plan to push this back to your own fork, commit
them on a branch of your own rather than on `main`, so pulling upstream fixes
stays a fast-forward:

```bash
git switch -c my-cluster
git commit -am "point manifests at $BASE_DOMAIN"
```

> [!CAUTION]
> Never commit the password files or certificates produced in steps 05 and 09.
> `.gitignore` already excludes the usual suspects, but the passwords live in a
> clone of a *different* repository — see [step 09](09-namespace-and-secrets.md).

## Done when

- `ls "$K8S"` lists 15 manifests
- No `yourdomain.com` or `192.168.1.50` remains in `k8s/`
- The current context's namespace is `eegfaktura`

→ [Step 05 — TLS](05-tls.md)
