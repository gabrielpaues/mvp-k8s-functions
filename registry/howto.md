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
- Docker Desktop on macOS

---

### Step 1: Configure Docker Desktop

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
kubectl port-forward svc/registry 5000:5000
```

Keep this terminal open.

---

### Step 4: Push an Image

In a new terminal:

```bash
# Pull test image
docker pull nginx:latest

# Tag for your registry
docker tag nginx:latest host.docker.internal:5000/nginx:latest

# Push
docker push host.docker.internal:5000/nginx:latest
```

---

### Step 5: Verify

```bash
# Check registry catalog
curl http://localhost:5000/v2/_catalog

# Pull the image back
docker rmi host.docker.internal:5000/nginx:latest
docker pull host.docker.internal:5000/nginx:latest

# Run it
docker run --rm -p 8080:80 host.docker.internal:5000/nginx:latest
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