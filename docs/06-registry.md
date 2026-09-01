# Step 06 — Container build tooling and a local registry

k3s uses **containerd**, which has no image *build* command. So Docker Engine
goes on as a build tool, the results are pushed to a local registry, and k3s
pulls from there.

This mirrors the upstream flow — build → registry → cluster — while staying
offline and fast. Real ghcr.io pushes come later, in [step 20](20-ci-pipeline.md).

## 6.1 Install Docker Engine

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

On Debian, replace both `ubuntu` occurrences with `debian`.

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin
```

Run Docker without `sudo`:

```bash
sudo usermod -aG docker "$USER"
newgrp docker
docker version
```

`newgrp` fixes the current shell; log out and back in for it to stick.

> [!NOTE]
> **Two containerds is fine.** Docker brings its own `containerd.io`; k3s runs
> its own embedded one. They do not conflict — separate sockets, separate image
> stores. That separation is exactly why a registry is needed to move images
> between them.

## 6.2 Run the registry

```bash
docker run -d --restart=always -p 5000:5000 \
  -v /var/lib/registry:/var/lib/registry \
  --name registry registry:2
```

```bash
curl -s http://localhost:5000/v2/_catalog
```

`{"repositories":[]}`

## 6.3 Point k3s at it

The registry serves plain HTTP, so containerd must be told that is expected:

```bash
sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/registries.yaml > /dev/null <<'EOF'
mirrors:
  "localhost:5000":
    endpoint:
      - "http://localhost:5000"
EOF
```

```bash
sudo systemctl restart k3s
```

> [!WARNING]
> **The restart is required.** `registries.yaml` is read only at startup.
> Skipping it produces `http: server gave HTTP response to HTTPS client` on the
> first pull, which reads like a registry fault rather than a config one.

## 6.4 Prove the loop

Build → push → pull into the cluster:

```bash
mkdir -p /tmp/hello && cd /tmp/hello
cat > Dockerfile <<'EOF'
FROM nginx:alpine
RUN echo "registry works" > /usr/share/nginx/html/index.html
EOF
```

```bash
docker build -t localhost:5000/hello:v1 .
docker push localhost:5000/hello:v1
```

```bash
curl -s http://localhost:5000/v2/_catalog
```

```bash
kubectl run hello --image=localhost:5000/hello:v1
kubectl wait --for=condition=Ready pod/hello --timeout=60s
kubectl exec hello -- curl -s localhost | head -1
kubectl delete pod hello
```

```bash
cd ~
```

## 6.5 The tagging convention

Every image in this guide is tagged `localhost:5000/<name>:dev`. One tag per
service keeps rebuilds trivial.

> [!IMPORTANT]
> Kubernetes caches by tag, so re-pushing `:dev` does **not** by itself cause a
> redeploy. Every manifest here sets `imagePullPolicy: Always` for that reason.
> After re-pushing an image, still run
> `kubectl rollout restart deployment/<name>` — the pod only re-pulls when it
> restarts.

## Done when

- `docker version` works without `sudo`
- `curl -s http://localhost:5000/v2/_catalog` lists `hello`
- The Pod reached `Ready`, proving k3s pulled from your registry

→ [Step 07 — language toolchains](07-toolchains.md)
