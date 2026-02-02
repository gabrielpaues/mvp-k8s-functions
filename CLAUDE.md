# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Kubernetes configuration project providing example deployments for:
- **NATS**: A publish-subscribe messaging system
- **Docker Registry**: A cluster-internal container registry

This is a configuration-only project with no build system—it uses kubectl to deploy prebuilt container images.

## Common Commands

### NATS Deployment
```bash
# Deploy NATS server
kubectl apply -f nats/nats.yaml

# Test pub-sub (requires two terminals)
kubectl apply -f nats/nats-sub-pod.yaml   # Terminal 1: subscriber
kubectl logs -f nats-sub
kubectl apply -f nats/nats-pub-pod.yaml   # Terminal 2: publisher
kubectl logs nats-pub

# Cleanup test pods
kubectl delete pod nats-sub nats-pub
```

### Registry Deployment
```bash
# Deploy registry
kubectl apply -f registry/registry.yaml
kubectl wait --for=condition=ready pod -l app=registry --timeout=60s

# Port forward for local access
kubectl port-forward svc/registry 5000:5000

# Push an image
docker tag nginx:latest host.docker.internal:5000/nginx:latest
docker push host.docker.internal:5000/nginx:latest

# Verify registry contents
curl http://localhost:5000/v2/_catalog
```

## Architecture

### Security Model
All workloads follow Kubernetes security best practices:
- Non-root containers (UID 1000)
- Seccomp RuntimeDefault profiles
- All Linux capabilities dropped
- Privilege escalation disabled
- NATS uses read-only root filesystem

### Services
| Service | Ports | Image |
|---------|-------|-------|
| NATS | 4222 (client), 8222 (monitoring) | nats:2.10-alpine |
| Registry | 5000 | registry:2 |

Both services use ClusterIP type, accessible only within the cluster. Use port-forwarding for local access.
