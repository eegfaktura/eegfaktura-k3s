# Step 07 — Language toolchains

The nine source-built services span five ecosystems. Installing everything now
makes every later step pure build-and-deploy.

| Toolchain | Needed by |
|---|---|
| Go 1.25 + protoc | backend, energystore |
| Node 22 | web, admin (frontends) |
| JDK 17 + sbt | admin-backend, eda-xp |
| JDK 17 + Maven | billing |
| Python 3 | filestore |

## 7.1 Go

The version must match `go.mod` (`go 1.25.0`) and the Dockerfile (`golang:1.25`).
Ubuntu's apt package lags, so install from upstream:

```bash
GO_VER=1.25.14
curl -LO "https://go.dev/dl/go${GO_VER}.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go${GO_VER}.linux-amd64.tar.gz"
rm "go${GO_VER}.linux-amd64.tar.gz"
```

```bash
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> ~/.bashrc
source ~/.bashrc
go version
```

## 7.2 protoc and the code generators

The backend **does not compile from a clean checkout** without these — you get
`undefined: protobuf.UpdateEegReply`. Go has no build hook, so `go build` never
runs `go generate` for you.

```bash
sudo apt-get install -y protobuf-compiler
protoc --version
```

```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
go install github.com/atombender/go-jsonschema@latest
```

```bash
ls ~/go/bin
```

`go-jsonschema`, `protoc-gen-go`, `protoc-gen-go-grpc`.

> [!NOTE]
> **`@latest` is not reproducible.** The upstream Dockerfile installs `@latest`
> too, so two builds on different days can emit different generated code — we
> observed `SupportPackageIsVersion7` become `9`. Acceptable for a dev cluster;
> a real reproducibility gap if you later need deterministic builds. See
> [known problems #8](known-problems.md#8-generated-protobuf-code-is-handled-three-ways).

## 7.3 Node 22

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs
node --version && npm --version
```

## 7.4 JDK 17

The Scala images are based on `eclipse-temurin:17-jre`, so build against 17.

```bash
sudo apt-get install -y openjdk-17-jdk
java -version
```

```bash
echo 'export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64' >> ~/.bashrc
source ~/.bashrc
```

## 7.5 sbt

```bash
echo "deb https://repo.scala-sbt.org/scalasbt/debian all main" \
  | sudo tee /etc/apt/sources.list.d/sbt.list
curl -sL "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x2EE0EA64E40A89B84B2DF73499E82A75642AC823" \
  | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/sbt.gpg
sudo apt-get update && sudo apt-get install -y sbt
```

```bash
sbt --version
```

The first run downloads a lot. Let it finish here rather than in the middle of
[step 16](16-admin-backend.md).

## 7.6 Maven and Python

```bash
sudo apt-get install -y maven python3 python3-pip python3-venv
mvn -version
python3 --version
```

## 7.7 Clone the source repos

```bash
mkdir -p ~/src && cd ~/src
for r in eegfaktura-backend eegfaktura-web eegfaktura-admin \
         eegfaktura-admin-backend eegfaktura-eda-xp eegfaktura-billing \
         eegfaktura-energystore eegfaktura-filestore eegfaktura-keycloak \
         eegfaktura-postgresql eegfaktura-mosquitto eegfaktura-postfix \
         eegfaktura-docker-compose; do
  git clone -q "https://github.com/eegfaktura/$r.git"
done
ls
```

```bash
cd ~
```

> [!NOTE]
> **Why `eegfaktura-docker-compose` is in that list.** It is the source of truth
> for every environment variable, secret, port and config file the manifests
> reproduce. From [step 09](09-namespace-and-secrets.md) onward you refer to it
> constantly, and several files are read straight out of it.

## Done when

```bash
go version && node --version && java -version && sbt --version && mvn -version && protoc --version
```

All report without error, and `~/src` holds the 13 repositories.

→ [Step 08 — infrastructure images](08-infra-images.md)
