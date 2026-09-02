# Update: `eegfaktura-admin`

|  |  |
|---|---|
| Source | `~/src/eegfaktura-admin` |
| Image | `localhost:5000/eeg-registration-frontend:dev` |
| Deployment | `eegfaktura-admin-web` |
| Container | `admin-web` |
| Manifest | [`k8s/71-admin-web.yaml`](../k8s/71-admin-web.yaml) |
| First built in | [step 18](../docs/18-frontends.md) |
| Deployed in | [step 18](../docs/18-frontends.md) |
| Rollout | RollingUpdate — the new pod becomes ready before the old one is removed, so there is no gap. |

The admin portal — Create React App, npm, and output in `build/` rather than `dist/`. Same shape as the web frontend: build on the VM first, then package. Note the deployment is `eegfaktura-admin-web`, not `eegfaktura-admin`.

## The whole update, in one paste

```bash
REF=main                       # a branch name, or a release tag
cd ~/src/eegfaktura-admin
git fetch --all --tags
git checkout "$REF"
git pull --ff-only 2>/dev/null || echo "(detached at a tag — nothing to pull)"
git log --oneline -1
```

```bash
cd ~/src/eegfaktura-admin
rm -rf build
npm install
TSC_COMPILE_ON_ERROR=true npm run build
ls build
docker build -t localhost:5000/eeg-registration-frontend:dev ~/src/eegfaktura-admin
docker push localhost:5000/eeg-registration-frontend:dev
kubectl rollout restart deployment/eegfaktura-admin-web
kubectl rollout status  deployment/eegfaktura-admin-web --timeout=300s
```

If any of that fails, the same thing broken into steps is below.

## 1. Fetch

```bash
cd ~/src/eegfaktura-admin
git status --short
```

Anything listed is dealt with before pulling — see
[the summary](README.md#1-fetch) for the two repos that dirty themselves.

```bash
git fetch --all --tags
git checkout "$REF"          # branch or tag
git pull --ff-only           # branches only; a tag is already fixed
git log --oneline -1
```

## 2. Build

First, on the VM:

```bash
cd ~/src/eegfaktura-admin
rm -rf build
npm install
TSC_COMPILE_ON_ERROR=true npm run build
ls build
```

Then the image:

```bash
docker build -t localhost:5000/eeg-registration-frontend:dev ~/src/eegfaktura-admin
```

## 3. Push

```bash
docker push localhost:5000/eeg-registration-frontend:dev
```

## 4. Roll out

```bash
kubectl rollout restart deployment/eegfaktura-admin-web
kubectl rollout status  deployment/eegfaktura-admin-web --timeout=300s
```

## 5. Verify

```bash
kubectl get pods -l app=eegfaktura-admin-web
```

```bash
kubectl exec deployment/eegfaktura-admin-web -- ls /var/www/html/registration-web/
```

Confirm the running pod is the image you just pushed, not a cached layer:

```bash
kubectl describe pod -l app=eegfaktura-admin-web | grep -E 'Image:|Image ID:'
```

## Rollback

`kubectl rollout undo` does **not** work here — the previous pod template names
the same mutable `:dev` tag, so it re-pulls the image you are trying to escape.
Tag the build you are replacing *before* you overwrite it:

```bash
SHA=$(git -C ~/src/eegfaktura-admin rev-parse --short HEAD)
docker tag  localhost:5000/eeg-registration-frontend:dev localhost:5000/eeg-registration-frontend:"$SHA"
docker push localhost:5000/eeg-registration-frontend:"$SHA"
```

Then going back is naming it:

```bash
kubectl set image deployment/eegfaktura-admin-web admin-web=localhost:5000/eeg-registration-frontend:<old-sha>
kubectl rollout status deployment/eegfaktura-admin-web --timeout=300s
```

## Watch out for

- **`npm install`, never `npm ci`.** The lockfile is out of sync with `package.json`; `npm ci` installs the floor of every caret range and that tree fails to type-check ([known problems #18](../docs/known-problems.md#18-eegfaktura-admins-lockfile-is-out-of-sync-with-packagejson)). `npm install` rewrites `package-lock.json` — expected, and the reason `git status` is dirty next time.
- **`TSC_COMPILE_ON_ERROR=true` is required**, not a convenience. The source carries pre-existing MUI type errors that Babel compiles correctly and ForkTsChecker rejects. Upstream's CI sets the same variable. Do not patch the source to silence it.
- **`rm -rf build` for the same reason as the web frontend** — a stale `build/` will be packaged and pushed without complaint.
- **Discard the lockfile before pulling.** `git checkout package-lock.json` first, or the next `git pull` stops on a conflict in a file you never edited.

---

[← all services](README.md)
