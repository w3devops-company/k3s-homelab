# Kubernetes Deployment, Ingress & Packaging Guide

A practical reference for deploying, exposing, and packaging applications on K3s—covering **Deployments**, **Ingress**, **Kustomize**, **Helm Charts**, and **CI/CD**.

---

## 1. How Kubernetes Deployments Work

A **Deployment** is a declarative controller for stateless workloads. You define the **desired state** in YAML, and Kubernetes continuously reconciles the **actual state** to match it.

### Object Hierarchy
```text
[ Deployment: web-demo ]            (Manages versions, rolling updates & rollbacks)
          │
          ▼
[ ReplicaSet: web-demo-7c87c97fb9 ] (Ensures exact replica count = 6)
          │
          ├──► [ Pod: web-demo-7c87c97fb9-4bq2p ]
          ├──► [ Pod: web-demo-7c87c97fb9-6fs9l ]
          └──► [ ... ]
```

- **Pod**: Smallest deployable unit wrapping container(s). Pods are ephemeral—if one crashes, it is replaced rather than repaired.
- **ReplicaSet**: Ensures the exact desired number of Pods are running at all times.
- **Deployment**: Manages ReplicaSets to enable zero-downtime updates and instant rollbacks.

### Pod Lifecycle on `kubectl apply`
1. **API Server** validates and stores the Deployment manifest.
2. **Deployment Controller** creates/updates a `ReplicaSet`.
3. **ReplicaSet Controller** creates `Pod` objects (initially `Pending`).
4. **Scheduler (`kube-scheduler`)** picks a node with enough `resources.requests` and no untolerated taints (e.g., `disk-pressure:NoSchedule`).
5. **Kubelet + `containerd`** pulls the container image and starts the container.
6. **Probes & Service** verify health and register the Pod IP into Service endpoints.

### Zero-Downtime Rolling Updates & Rollbacks
When you update an image or container spec:
- Kubernetes creates a **new ReplicaSet (v2)** and scales it up gradually (`maxSurge` / `maxUnavailable`) while scaling down the **old ReplicaSet (v1)**.
- Traffic only shifts to new pods once their **`readinessProbe`** passes.
- Old ReplicaSets are retained at `0` replicas so rollbacks are instant:

```bash
# Watch rollout progress
kubectl rollout status deployment/web-demo

# View revision history
kubectl rollout history deployment/web-demo

# Roll back to previous working revision
kubectl rollout undo deployment/web-demo

# Restart all pods cleanly (rolling restart)
kubectl rollout restart deployment/web-demo
```

### Health Checks (`readinessProbe` & `livenessProbe`)
- **`readinessProbe`**: Controls **traffic routing**. If it fails, the Pod is removed from Service endpoints until healthy.
- **`livenessProbe`**: Controls **container restarts**. If it fails repeatedly, `kubelet` kills and restarts the container.

---

## 2. Ingress (HTTP/HTTPS Routing)

While `NodePort` exposes an app on a high port (`30080`), **Ingress** provides a smart Layer-7 reverse proxy on standard ports **`80` (HTTP)** and **`443` (HTTPS)** with domain and path routing.

### Architecture
```text
[ Browser / Client ]
       │  (http://web.local or https://app.example.com)
       ▼
[ Ingress Controller: Traefik ]  (Pre-installed in K3s on ports 80 & 443)
       │
       ├── host: web.local      ──► [ Service: web-demo (ClusterIP:80) ]  ──► Nginx Pods
       ├── host: grafana.local  ──► [ Service: prometheus-grafana (:80) ] ──► Grafana Pod
       └── path: /api           ──► [ Service: backend-api (:8080) ]      ──► API Pods
```

- **Ingress Controller**: The proxy engine running in the cluster. K3s includes **Traefik** out of the box in `kube-system`.
- **Ingress Resource**: The YAML rules telling Traefik how to route incoming requests to internal `ClusterIP` Services.

### Example Ingress Manifest
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-demo-ingress
  namespace: default
spec:
  rules:
  - host: web.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-demo
            port:
              number: 80
```

### Testing Ingress Locally
```bash
# Inspect Ingress resources
kubectl get ingress

# Test without editing /etc/hosts by passing the Host header
curl -H "Host: web.local" http://localhost
```
*(Or add `127.0.0.1 web.local` to `/etc/hosts` and open `http://web.local` in your browser.)*

---

## 3. Packaging & Environment Management: YAML vs. Kustomize vs. Helm

| Feature | Plain YAML | Kustomize (`kubectl -k`) | Helm Charts |
|---|---|---|---|
| **Best For** | Learning & single env | Multi-env overlays (`dev`/`prod`) | Complex apps, 3rd-party tools, CI/CD |
| **Templating** | None | Patching / Overlays (template-free) | Go templates (`{{ .Values... }}`) |
| **Built into `kubectl`?** | Yes | Yes (`kubectl apply -k`) | No (uses `helm` CLI) |
| **Release Tracking / Rollback** | Manual / Deployment history | Manual / Deployment history | Built-in (`helm rollback`, `--atomic`) |
| **Complexity** | Low | Medium | Higher |

---

### Option A: Plain YAML
Best when learning or managing 1–5 manifests in a single environment. What you read in the file is exactly what runs in Kubernetes.

```bash
kubectl apply -f nginx/nginx-deployment.yaml
```

---

### Option B: Kustomize (Template-Free Overlays)
Built directly into `kubectl`. Keep standard YAML in `base/`, and override specific fields (like `replicas` or `image` tags) per environment in `overlays/`:

```text
app/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml   # e.g., replicas: 1
    └── prod/
        └── kustomization.yaml   # e.g., replicas: 6
```

Example `overlays/prod/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
replicas:
  - name: web-demo
    count: 6
images:
  - name: nginx
    newTag: "1.27-alpine"
```

```bash
# Preview rendered YAML before applying
kubectl kustomize app/overlays/prod

# Apply directly
kubectl apply -k app/overlays/prod
```

---

### Option C: Helm Charts (Kubernetes Package Manager)
Helm bundles multiple Kubernetes resources into a reusable, versioned package driven by `values.yaml`.

#### Chart Structure
```text
my-web-chart/
├── Chart.yaml          # Chart name, version, description
├── values.yaml         # Default configuration values
└── templates/
    ├── deployment.yaml # Uses {{ .Values.replicas }}, {{ .Values.image.tag }}
    ├── service.yaml
    └── ingress.yaml
```

#### Essential Helm Commands
```bash
# Repositories
helm repo add <name> <url>
helm repo update

# Download a chart archive locally (without installing)
helm pull <repo>/<chart> --untar

# Install or upgrade a release (idempotent)
helm upgrade --install <release-name> <chart> \
  --namespace <ns> --create-namespace \
  -f values-prod.yaml \
  --set image.tag=1.2.3 \
  --atomic

# List installed releases
helm list -A

# View release history & roll back
helm history <release-name> -n <ns>
helm rollback <release-name> <revision> -n <ns>

# Uninstall a release
helm uninstall <release-name> -n <ns>
```

---

## 4. Recommended CI/CD Pattern for Web Apps

```text
[ git push / PR ]
       │
       ▼
[ CI: GitHub Actions ]
  1. Run linter & unit tests
  2. Validate K8s YAML (kubeconform / kubectl --dry-run=client)
  3. Build Docker image tagged with Git SHA (e.g. :a1b2c3d)
  4. Scan image for vulnerabilities (Trivy)
  5. Push image to registry (ghcr.io)
       │
       ▼
[ CD: Deploy to K3s ]
  • GitOps (ArgoCD / Flux): Cluster pulls updated manifests from Git (no open ports needed)
  • OR Self-Hosted Runner: Runs `kubectl set image` / `helm upgrade` locally
```

### Which Should You Use in `k3s-homelab`?
- **3rd-Party Infrastructure** (Prometheus, Grafana, ArgoCD, PostgreSQL) → **Helm**
- **Your Own Web Apps (Learning / Single Cluster)** → **Plain YAML** or **Kustomize**
- **Multi-Environment Apps / Reusable Internal Packages** → **Helm Chart**
