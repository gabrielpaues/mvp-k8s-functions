# MVP Kubernetes Functions

Example Kubernetes deployments for a container registry and NATS message queue, configured with security best practices.

## Prerequisites

- **Kubernetes cluster** with PodSecurity "restricted" enforcement
- **kubectl** configured to access your cluster
- **Docker Desktop** or **Podman** (for local development)

### Docker Desktop Configuration

For the registry to work with Docker Desktop, configure insecure registries:

1. Open Docker Desktop → Settings → Docker Engine
2. Add `"insecure-registries": ["host.docker.internal:5000"]` to the JSON config
3. Click **Apply & Restart**

### Podman on macOS

On macOS, Podman runs containers inside a Linux VM (Podman machine).

1. Install and start:

```bash
brew install podman
podman machine init
podman machine start
```

2. Edit (or create) `~/.config/containers/registries.conf`:

```toml
[[registry]]
location = "host.containers.internal:5000"
insecure = true
```

3. Restart the machine to pick up the config:

```bash
podman machine stop
podman machine start
```

**Networking note:** The Podman machine VM cannot reach the macOS host via `localhost`. Use `host.containers.internal` to reference the host. The port-forward must also bind to all interfaces so the VM can reach it:

```bash
kubectl port-forward --address 0.0.0.0 svc/registry 5000:5000
```

### Podman on Linux

On Linux, Podman runs natively — no VM is needed.

1. Install Podman via your package manager (e.g. `sudo apt install podman` or `sudo dnf install podman`)

2. Edit (or create) `~/.config/containers/registries.conf`:

```toml
[[registry]]
location = "localhost:5000"
insecure = true
```

## Setup

### 1. Deploy the Container Registry

```bash
kubectl apply -f registry/registry.yaml
kubectl wait --for=condition=ready pod -l app=registry --timeout=60s
```

Start port forwarding (keep this terminal open):

```bash
# Docker Desktop or Podman on Linux
kubectl port-forward svc/registry 5000:5000

# Podman on macOS (bind to all interfaces so the VM can reach the host)
kubectl port-forward --address 0.0.0.0 svc/registry 5000:5000
```

Push an image to the registry:

```bash
# Using Docker Desktop
docker pull nginx:latest
docker tag nginx:latest host.docker.internal:5000/nginx:latest
docker push host.docker.internal:5000/nginx:latest

# Using Podman on macOS
podman pull nginx:latest
podman tag nginx:latest host.containers.internal:5000/nginx:latest
podman push --tls-verify=false host.containers.internal:5000/nginx:latest

# Using Podman on Linux
podman pull nginx:latest
podman tag nginx:latest localhost:5000/nginx:latest
podman push --tls-verify=false localhost:5000/nginx:latest
```

Verify:

```bash
curl http://localhost:5000/v2/_catalog
```

See [registry/howto.md](registry/howto.md) for the complete walkthrough.

### 2. Deploy NATS Message Queue

```bash
kubectl apply -f nats/nats.yaml
kubectl wait --for=condition=ready pod -l app=nats --timeout=60s
```

Test pub-sub messaging (requires two terminals):

```bash
# Terminal 1 - Start subscriber
kubectl apply -f nats/nats-sub-pod.yaml
kubectl logs -f nats-sub

# Terminal 2 - Publish message
kubectl apply -f nats/nats-pub-pod.yaml
kubectl logs nats-pub
```

Cleanup test pods:

```bash
kubectl delete pod nats-sub nats-pub
```

## Cleanup

Remove all deployments:

```bash
kubectl delete -f registry/registry.yaml
kubectl delete -f nats/nats.yaml
kubectl delete pod nats-sub nats-pub --ignore-not-found
```

## License

GNU General Public License v3 - see [LICENSE](LICENSE) for details.
