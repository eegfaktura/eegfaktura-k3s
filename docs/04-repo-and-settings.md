# Step 04 — The repo and your settings

The manifests are real files in this repository, not snippets to copy out of
markdown. Clone it onto the VM once, render your hostnames into it, and every
later step applies files from `$MANIFESTS`.

## 4.1 Clone

```bash
cd ~
git clone https://github.com/<owner>/eegfaktura-k3s.git eegfaktura-k3s
cd ~/eegfaktura-k3s
ls k8s/
```

Use your own fork if you intend to keep changes (see [4.5](#45-keeping-your-changes)),
or clone this repository directly to follow along.

> [!NOTE]
> The target directory is named explicitly, because `$MANIFESTS` points at
> `~/eegfaktura-k3s/k8s.local`. If your fork or mirror has a different
> repository name, this keeps the path right without editing `~/.bashrc`.

```bash
ls k8s/*.yaml | wc -l      # 15
```

## 4.2 What is in here

```
docs/        this guide, one file per step, plus known-problems.md
k8s/         manifest templates — tracked in git, never edited
k8s.local/   the rendered copies you apply — generated, git-ignored
```

The manifest numbers are independent of the step numbers; they follow the
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

## 4.3 Render the manifests

Seven of the fifteen manifests reference your domain or address, as
`${BASE_DOMAIN}` and `${VM_IP}`:

```bash
grep -rl 'BASE_DOMAIN\|VM_IP' k8s/
```

`envsubst` expands them into `k8s.local/`, which is what `$MANIFESTS` points at:

```bash
cd ~/eegfaktura-k3s
mkdir -p "$MANIFESTS"
for f in k8s/*.yaml; do
  envsubst '$BASE_DOMAIN $VM_IP' < "$f" > "$MANIFESTS/$(basename "$f")"
done
```

> [!IMPORTANT]
> **The quoted variable list is not optional.** Bare `envsubst` expands *every*
> `${...}` it sees, including `${MAIL_HOST}` in a comment in `60-billing.yaml`
> that documents why billing refuses to boot without it — it would be silently
> replaced with nothing. Naming the two variables confines the substitution to
> them. The single quotes keep the shell from expanding the list before
> `envsubst` reads it.

Check the result:

```bash
grep -rn 'BASE_DOMAIN\|VM_IP' "$MANIFESTS"/
```

```bash
grep -rho "[a-z]*\.$BASE_DOMAIN" "$MANIFESTS"/ | sort -u
```

The first must print **nothing** — an unexpanded `${BASE_DOMAIN}` means the
variable was not set when you rendered. The second must print exactly your three
hostnames; blank output, or names beginning with a bare dot, means
`BASE_DOMAIN` was empty.

> [!TIP]
> **Re-render after `git pull`, and after any change to `BASE_DOMAIN` or
> `VM_IP`.** It is idempotent, so re-running it is always safe:
> ```bash
> cd ~/eegfaktura-k3s && for f in k8s/*.yaml; do envsubst '$BASE_DOMAIN $VM_IP' < "$f" > "$MANIFESTS/$(basename "$f")"; done
> ```
> Rendering alone changes nothing in the cluster — re-apply the affected
> manifest to make a change take effect.

<details>
<summary>Why render, rather than edit the manifests in place</summary>

Editing `k8s/` directly works, and for a one-off cluster it is perfectly fine.
Rendering buys three things:

- **`git pull` stays a fast-forward.** In-place edits touch seven tracked files,
  so every upstream change becomes a merge conflict in files you did not mean to
  own.
- **One source of truth.** The domain exists in `~/.bashrc` and nowhere else. In
  the in-place approach it lives in the shell *and* in the manifests, and the two
  can drift — usually noticed as a Keycloak issuer mismatch three steps later.
- **`k8s.local/` is git-ignored**, so a rendered file can never be committed with
  your real hostnames in it.

</details>

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

```bash
git status --short
```

This should show **nothing but** `k8s.local/` being ignored — the tracked
templates are untouched, which is the point of 4.3. If you do change a template
(different resource limits, an extra service), commit it on a branch of your own
so pulling upstream fixes stays a fast-forward:

```bash
git switch -c my-cluster
```

> [!CAUTION]
> Never commit the password files or certificates produced in steps 05 and 09.
> `.gitignore` covers `k8s.local/`, keys and certificates here — but the
> passwords live in a clone of a *different* repository, so see
> [step 09](09-namespace-and-secrets.md).

## Done when

- `ls "$MANIFESTS"` lists 15 rendered manifests
- No `${BASE_DOMAIN}` or `${VM_IP}` remains unexpanded in `$MANIFESTS`
- The current context's namespace is `eegfaktura`

→ [Step 05 — TLS](05-tls.md)
