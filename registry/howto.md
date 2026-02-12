Let's tear it all down:

```bash
# Delete registry
kubectl delete deployment registry
kubectl delete service registry

# Delete any test pods
kubectl delete pod nats-sub nats-pub push-test --ignore-not-found
```

---

## Complete Walkthrough: Container Registry on Kubernetes

### Prerequisites
- Kubernetes cluster with PodSecurity "restricted" enforcement
- kubectl configured
- Docker Desktop or Podman

---

### Step 1: Configure Your Container Runtime

#### Option A: Docker Desktop

Open Docker Desktop → Settings → Docker Engine, and set:

```json
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "experimental": false,
  "insecure-registries": ["host.docker.internal:5000"]
}
```

Click **Apply & Restart**.

#### Option B: Podman on macOS

On macOS, Podman runs containers inside a Linux VM (Podman machine).

Install and start:

```bash
brew install podman
podman machine init
podman machine start
```

Edit (or create) `~/.config/containers/registries.conf`:

```toml
[[registry]]
location = "host.containers.internal:5000"
insecure = true
```

Restart the machine to pick up the config:

```bash
podman machine stop
podman machine start
```

> **Networking note:** The Podman machine VM cannot reach the macOS host via
> `localhost`. Use `host.containers.internal` to reference the host, and bind
> port-forward to all interfaces (see Step 3).

#### Option C: Podman on Linux

On Linux, Podman runs natively — no VM is needed.

Install via your package manager (e.g. `sudo apt install podman` or `sudo dnf install podman`).

Edit (or create) `~/.config/containers/registries.conf`:

```toml
[[registry]]
location = "localhost:5000"
insecure = true
```

---

### Step 2: Deploy the Registry

Create `registry.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: registry
  template:
    metadata:
      labels:
        app: registry
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: registry
        image: registry:2
        ports:
        - containerPort: 5000
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
        volumeMounts:
        - name: registry-data
          mountPath: /var/lib/registry
      volumes:
      - name: registry-data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: registry
spec:
  selector:
    app: registry
  ports:
  - port: 5000
```

Deploy it:

```bash
kubectl apply -f registry.yaml
kubectl wait --for=condition=ready pod -l app=registry --timeout=60s
```

---

### Step 3: Start Port Forward

```bash
# Docker Desktop or Podman on Linux
kubectl port-forward svc/registry 5000:5000

# Podman on macOS (bind to all interfaces so the VM can reach the host)
kubectl port-forward --address 0.0.0.0 svc/registry 5000:5000
```

Keep this terminal open.

---

### Step 4: Push an Image

In a new terminal:

**Docker:**

```bash
docker pull nginx:latest
docker tag nginx:latest host.docker.internal:5000/nginx:latest
docker push host.docker.internal:5000/nginx:latest
```

**Podman on macOS:**

```bash
podman pull nginx:latest
podman tag nginx:latest host.containers.internal:5000/nginx:latest
podman push --tls-verify=false host.containers.internal:5000/nginx:latest
```

**Podman on Linux:**

```bash
podman pull nginx:latest
podman tag nginx:latest localhost:5000/nginx:latest
podman push --tls-verify=false localhost:5000/nginx:latest
```

---

### Step 5: Verify

```bash
# Check registry catalog
curl http://localhost:5000/v2/_catalog
```

**Docker:**

```bash
docker rmi host.docker.internal:5000/nginx:latest
docker pull host.docker.internal:5000/nginx:latest
docker run --rm -p 8080:80 host.docker.internal:5000/nginx:latest
```

**Podman on macOS:**

```bash
podman rmi host.containers.internal:5000/nginx:latest
podman pull --tls-verify=false host.containers.internal:5000/nginx:latest
podman run --rm -p 8080:80 host.containers.internal:5000/nginx:latest
```

**Podman on Linux:**

```bash
podman rmi localhost:5000/nginx:latest
podman pull --tls-verify=false localhost:5000/nginx:latest
podman run --rm -p 8080:80 localhost:5000/nginx:latest
```

Test in browser or with curl:

```bash
curl http://localhost:8080
```

You should see the nginx welcome page.

---

### Summary

| Component | Purpose |
|-----------|---------|
| `registry.yaml` | Deploys registry with restricted PodSecurity |
| `emptyDir` volume | Provides writable storage for registry |
| `port-forward` | Tunnels local port 5000 to cluster |
| `host.docker.internal` | Allows Docker Desktop VM to reach host's port-forward |
| `host.containers.internal` | Allows Podman machine VM (macOS) to reach host's port-forward |
| `localhost:5000` | Registry address when using Podman on Linux |
| `--address 0.0.0.0` | Binds port-forward to all interfaces (required for Podman on macOS) |