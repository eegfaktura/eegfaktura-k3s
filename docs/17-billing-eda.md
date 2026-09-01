# Step 17 — Billing and EDA (JVM)

The two remaining JVM services.

Manifests: **`k8s/60-billing.yaml`**, **`k8s/61-eda.yaml`**

> [!NOTE]
> **Both are optional.** Neither is needed for registration or the master-data
> workflow that [step 19](19-bootstrap-and-verify.md) verifies. Billing is
> invoicing only — upstream already removed it from the `depends_on` lists of
> `eegfaktura-web` and `eegfaktura-proxy`. EDA has no working Ponton gateway in
> this stack. Skipping this step costs nothing for step 19 and saves two JVM
> heaps, which matters on a memory-constrained host. You can come back to it
> later.

## 17.1 Build billing

It has a Dockerfile, so this is a plain build:

```bash
docker build -t localhost:5000/eegfaktura-billing:dev ~/src/eegfaktura-billing
docker push localhost:5000/eegfaktura-billing:dev
```

## 17.2 Build EDA

Scala via sbt-native-packager, same path as the admin backend. The source repo
is `eegfaktura-eda-xp`; the **image is named `eegfaktura-kep`**.

```bash
cd ~/src/eegfaktura-eda-xp
sbt Docker/publishLocal
```

> [!NOTE]
> **`[error]`-prefixed output is not an error.** sbt routes BuildKit's stderr
> progress through its error channel, so the whole docker build appears as
> `[error]` lines. Read the step results: if each stage says `DONE` and the run
> ends in `[success]`, it worked.

```bash
docker images | grep -i kep
```

```bash
KEP_TAG=$(docker images --format '{{.Tag}}' ghcr.io/vfeeg-development/eegfaktura-kep | grep -v latest | head -1)
echo "$KEP_TAG"
```

```bash
docker tag ghcr.io/vfeeg-development/eegfaktura-kep:"$KEP_TAG" localhost:5000/eegfaktura-kep:dev
docker push localhost:5000/eegfaktura-kep:dev
```

```bash
cd ~
```

## 17.3 Create the EDA config

EDA reads a HOCON config that compose bind-mounts at `/conf/application.conf`.
It must come from your **working copy**, which carries the regenerated database
password from [step 9.1](09-namespace-and-secrets.md#91-regenerate-the-passwords):

```bash
kubectl create configmap eda-config --from-file=application.conf="$HOME/src/eegfaktura-docker-compose/eegfaktura-eda.application.conf"
```

```bash
kubectl get configmap eda-config -o jsonpath='{.data.application\.conf}' | grep -c 'Dzy5lShLn1N3rqTM'
```

> [!CAUTION]
> **This must print `0`.** A `1` means the ConfigMap carries the *committed*
> password while the database uses your regenerated one, and EDA will fail to
> connect. Go back and apply the `sed` from step 9.1, then delete and recreate
> this ConfigMap.

## 17.4 Apply

```bash
kubectl apply -f "$K8S"/60-billing.yaml -f "$K8S"/61-eda.yaml
```

```bash
kubectl rollout status deployment/eegfaktura-billing --timeout=300s
kubectl rollout status deployment/eegfaktura-eda     --timeout=300s
```

Wait for the rollouts before reading logs — a pod that has not started yet has
none.

## 17.5 Verify

```bash
kubectl logs deployment/eegfaktura-billing --tail=30 | grep -E "Started|Tomcat|ERROR"
```

Success is `Started EegfakturaBillingApplication in …`. Billing owns the
`billingj` schema and migrates it with Flyway on first boot — separate from
`base`, so it does not hit the ownership conflict from
[step 10.4](10-postgres.md#104-hand-the-seeded-schemas-to-their-owners).

> [!CAUTION]
> **`MAIL_HOST` is mandatory.** `docker-compose.yaml` never defines it, and
> Spring evaluates it during context startup, so its absence is a hard boot
> failure rather than a warning:
> ```
> PlaceholderResolutionException: Could not resolve placeholder 'MAIL_HOST' in value "${MAIL_HOST}"
> ```
> The manifest sets `MAIL_HOST=eegfaktura-postfix` and `MAIL_PORT=25`.
> [Known problems #3](known-problems.md#3-mail_host-is-never-defined).

```bash
kubectl logs deployment/eegfaktura-eda --tail=30
```

```bash
curl -s -o /dev/null -w '%{http_code}\n' "https://$APP_HOST/cash/"
```

Anything other than a bare 404 means the route works.

> [!NOTE]
> **EDA cannot do real market communication here.** `kepserver.url` points at an
> unused `localhost:6060` and the mail-based path has no credentials. The
> service runs, but external EDA exchange is inert — which is why
> [step 19](19-bootstrap-and-verify.md) sets *Ponton Kommunikation* to
> "ignorieren", and why metering points stay in `INIT`/`NEW` rather than
> reaching `ACTIVE`.

## 17.6 Watch memory

Two JVM heaps are a meaningful addition on a constrained host:

```bash
kubectl top pods --sort-by=memory
```

Both manifests cap at `1536Mi`. If pods are OOM-killed or evicted, lower those
limits, or deploy the two services one at a time to see the real cost.

## Done when

- Billing logs `Started EegfakturaBillingApplication`
- EDA is `Running` and not crash-looping
- `/cash` returns something other than a bare 404

→ [Step 18 — frontends](18-frontends.md)
