# MVP Kubernetes Functions

Example Kubernetes deployments for a container registry and NATS message queue, configured with security best practices.

## Prerequisites

- **Kubernetes cluster** with PodSecurity "restricted" enforcement
- **kubectl** configured to access your cluster
- **Docker Desktop** (for local development on macOS/Windows)

### Docker Desktop Configuration

For the registry to work with Docker Desktop, configure insecure registries:

1. Open Docker Desktop → Settings → Docker Engine
2. Add `"insecure-registries": ["host.docker.internal:5000"]` to the JSON config
3. Click **Apply & Restart**

## Setup

### 1. Deploy the Container Registry

```bash
kubectl apply -f registry/registry.yaml
kubectl wait --for=condition=ready pod -l app=registry --timeout=60s
```

Start port forwarding (keep this terminal open):

```bash
kubectl port-forward svc/registry 5000:5000
```

Push an image to the registry:

```bash
docker pull nginx:latest
docker tag nginx:latest host.docker.internal:5000/nginx:latest
docker push host.docker.internal:5000/nginx:latest
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
