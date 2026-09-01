# Step 08 — Build the infrastructure images

Start with the four plain-Dockerfile images. No codegen, no JVM, no npm — they
prove the build → push → deploy loop before any hard service is involved.

## 8.1 Build and push all four

```bash
cd ~/src
for img in eegfaktura-postgresql eegfaktura-mosquitto eegfaktura-postfix eegfaktura-keycloak; do
  echo "=== building $img ==="
  docker build -t "localhost:5000/$img:dev" "./$img" || { echo "FAILED: $img"; break; }
  docker push "localhost:5000/$img:dev"
done
```

```bash
curl -s http://localhost:5000/v2/_catalog | jq
```

All four must appear.

```bash
cd ~
```

## 8.2 What each one is

| Image | Base | Role |
|---|---|---|
| `eegfaktura-postgresql` | postgres | init scripts create **both** the `eegfaktura` and `keycloak` databases and their users |
| `eegfaktura-mosquitto` | eclipse-mosquitto | MQTT broker between backend, energystore and eda |
| `eegfaktura-postfix` | debian + postfix | outbound mail relay; billing refuses to start without a mail host |
| `eegfaktura-keycloak` | `quay.io/keycloak/keycloak:26.4.7` | multi-stage: bakes in the custom themes and the realm export |

> [!NOTE]
> **Keycloak is a two-stage build.** The first stage runs `kc.sh build` to
> produce an optimised distribution, which is why the runtime command is
> `start --optimized`. It takes a few minutes — it is not hung.

## 8.3 The Postgres image matters more than it looks

The stack runs **one** Postgres serving two databases, created by the image's
init scripts from these variables:

```
POSTGRES_USER=postgres            # superuser
DB_USERNAME=eegfaktura            # app user  -> database "eegfaktura"
KEYCLOAK_DB_USERNAME=keycloak     # kc user   -> database "keycloak"
POSTGRES_INITDB_ARGS=--locale=de_DE:UTF8
```

> [!CAUTION]
> **Init scripts run once, on an empty data directory.** If the PVC already
> holds a cluster, changing these variables does nothing. Getting them wrong
> means deleting the PVC and starting over — so the passwords must be final
> before [step 10](10-postgres.md) first boots.

<details>
<summary>The German locale is recorded but inert — measured</summary>

`initdb` accepts `de_DE:UTF8` and the catalog reports it, but the image ships no
locale data:

```
The database cluster will be initialized with locale "de_DE:UTF8".
sh: locale: not found
WARNING:  no usable system locales were found
```

So collation falls back to byte order. Measured inside the image:

```
SELECT x FROM (VALUES ('Zeta'),('Ölberg'),('Apfel'),('Uhr')) t(x) ORDER BY x;
-->  Apfel, Uhr, Zeta, Ölberg        (byte order: Ö sorts after Z)
-->  Apfel, Ölberg, Uhr, Zeta        (what real de_DE would give)
```

**Keep the setting anyway** — the compose stack runs the same image with the
same argument, so reproducing it preserves parity. Just do not expect German
sort order: any name list ordered in the database, Excel exports included, puts
umlauts after `Z`. That is upstream behaviour, not something your cluster
introduced. See
[known problems #7](known-problems.md#7-postgres-declares-a-locale-it-cannot-provide).

</details>

## 8.4 Sanity-check the Postgres image (optional)

Faster to catch a broken build here than as a `CrashLoopBackOff` in step 10:

```bash
docker run --rm -d --name pgtest \
  -e POSTGRES_PASSWORD=test -e DB_USERNAME=eegfaktura -e DB_PASSWORD=test \
  -e DB_DATABASE=eegfaktura -e KEYCLOAK_DB_USERNAME=keycloak \
  -e KEYCLOAK_DB_PASSWORD=test -e KEYCLOAK_DB_DATABASE=keycloak \
  localhost:5000/eegfaktura-postgresql:dev
```

```bash
sleep 15 && docker exec pgtest psql -U postgres -l | grep -E "eegfaktura|keycloak"
```

```bash
docker rm -f pgtest
```

Both databases listed means the image works.

> [!NOTE]
> **The locale column reads `en_US.utf8` here, not the `de_DE:UTF8` of
> [8.3](#83-the-postgres-image-matters-more-than-it-looks).** This throwaway
> container is not given `POSTGRES_INITDB_ARGS`, so `initdb` falls back to the
> image's own `LANG`. Step 10 passes the argument and the catalog there reads
> `de_DE:UTF8` instead. Nothing to reconcile: collation is byte order in both
> cases, because the locale data is missing either way. Add
> `-e POSTGRES_INITDB_ARGS=--locale=de_DE:UTF8` to the `docker run` above if you
> want the check to mirror step 10 exactly.

## Done when

- Four images in `http://localhost:5000/v2/_catalog`
- The Postgres image creates both databases

→ [Step 09 — namespace, secrets and config](09-namespace-and-secrets.md)
