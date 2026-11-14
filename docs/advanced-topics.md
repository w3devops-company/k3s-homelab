# Kubernetes Advanced & Day-2 Production Topics

Once you have mastered core workloads and observability, these **Day-2 & Advanced** topics turn a K3s cluster into a full production platform—covering **Centralized Logging**, **GitOps**, **Automated TLS**, **Operators/CRDs**, **Advanced Scheduling**, and **AI/GPU Workloads**.

---

## 1. Centralized Logging (`Loki` + `Promtail` + Grafana)

- **What it is**: A log aggregation stack where **Promtail** (running as a `DaemonSet` on every node) collects container logs and ships them to **Loki**, which you query inside **Grafana**.
- **Why we use it**: `kubectl logs` only shows logs from *currently existing* pods. When a pod crashes, scales down, or is replaced, its logs are lost. Loki keeps historical logs searchable alongside your Prometheus metrics in Grafana.

### Architecture
```text
[ Pod stdout/stderr ] ──► [ Promtail (DaemonSet) ] ──► [ Loki ] ──► [ Grafana Explore ]
```

### Quick Setup on K3s (Helm)
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm upgrade --install loki grafana/loki-stack \
  --namespace monitoring \
  --set promtail.enabled=true \
  --set grafana.enabled=false   # Reuse your existing Grafana from kube-prometheus-stack
```

Then in Grafana:
1. Go to **Connections → Data sources → Add data source → Loki**.
2. Set URL to `http://loki.monitoring.svc.cluster.local:3100`.
3. Open **Explore** and query with **LogQL**:
   ```logql
   {namespace="default", app="web-demo"} |= "error"
   ```

---

## 2. GitOps Continuous Delivery (`ArgoCD`)

- **What it is**: A Kubernetes controller running *inside* your cluster that continuously watches a Git repository and automatically applies changes to match Git.
- **Why we use it**:
  - **No exposed ports**: Your homelab K3s API (`6443`) stays private behind your router—ArgoCD *pulls* from GitHub over outbound HTTPS.
  - **Drift detection**: If someone manually edits a live resource with `kubectl edit`, ArgoCD detects the drift and can auto-heal it back to what's in Git.
  - **Audit trail**: Every deployment change is a Git commit.

### Architecture
```text
[ GitHub Repo: k3s-homelab ] ◄──(HTTPS Pull every 3m)── [ ArgoCD (in K3s) ] ──► [ Syncs Pods/Services ]
```

### Quick Setup & Example `Application`
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Example ArgoCD `Application` manifest watching your `nginx/` folder:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/w3devops-company/k3s-homelab.git
    targetRevision: main
    path: nginx
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## 3. Automated HTTPS / TLS (`cert-manager` + Let's Encrypt)

- **What it is**: A Kubernetes controller that automatically provisions, attaches, and renews free SSL/TLS certificates (e.g., from Let's Encrypt) for your `Ingress` resources.
- **Why we use it**: Eliminates manual certificate generation and expiration outages; enables `https://` across all services via Traefik.

### How It Works with Ingress
1. Install `cert-manager` via Helm into `cert-manager` namespace.
2. Create a `ClusterIssuer` (for Let's Encrypt or a local self-signed CA).
3. Add an annotation and `tls:` block to your `Ingress`:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: web-demo-ingress
     annotations:
       cert-manager.io/cluster-issuer: letsencrypt-prod
   spec:
     tls:
     - hosts:
       - web.example.com
       secretName: web-demo-tls   # cert-manager auto-creates & renews this Secret
     rules:
     - host: web.example.com
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

---

## 4. Custom Resource Definitions (`CRDs`) & Operators

- **What a `CRD` is**: An extension to the Kubernetes API that lets you define your own resource types (just like `Pod` or `Deployment`). You already used two CRDs: **`PrometheusRule`** and **`AlertmanagerConfig`**.
- **What an `Operator` is**: A custom controller that watches CRDs and automates complex operational tasks (backups, failover, upgrades).
- **Why we use them**: Running production databases (like HA PostgreSQL) manually with raw `StatefulSet` YAML is hard. An Operator like **CloudNativePG** automates primary/replica failover, point-in-time recovery, and rolling upgrades:
  ```yaml
  apiVersion: postgresql.cnpg.io/v1
  kind: Cluster
  metadata:
    name: prod-postgres
  spec:
    instances: 3          # 1 Primary + 2 Streaming Replicas
    storage:
      size: 10Gi
  ```

---

## 5. Advanced Scheduling & Availability

| Concept | What It Is | Why / Where Used |
|---|---|---|
| **`DaemonSet`** | Ensures **1 pod runs on every node** in the cluster | Node-level agents (`node-exporter`, `promtail`, `svclb-traefik`) |
| **`Taints` & `Tolerations`** | Node repels pods unless the pod has a matching toleration | Dedicating GPU/high-memory nodes only to AI/DB workloads; cordoning unhealthy nodes (`disk-pressure`) |
| **`NodeAffinity`** | Rules attracting pods to specific node labels (`gpu=true`) | Placing AI inference pods on GPU nodes or fast-NVMe nodes |
| **`PodDisruptionBudget` (PDB)** | Guarantees a minimum number/percentage of pods stay `Ready` during voluntary disruptions | Prevents `kubectl drain` or node upgrades from taking down all replicas at once |

Example `PodDisruptionBudget` for `web-demo`:
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-demo-pdb
spec:
  minAvailable: 4   # At least 4 of 6 replicas must stay up during node maintenance
  selector:
    matchLabels:
      app: web-demo
```

---

## 6. AI & LLM Inference Workloads on K3s

*(Aligned with your `README.md` goal: hosting production AI models)*

- **GPU Support**: Install the **NVIDIA GPU Operator** (or `nvidia-device-plugin` DaemonSet) so pods can request GPUs:
  ```yaml
  resources:
    limits:
      nvidia.com/gpu: 1
  ```
- **Model Storage (`PVC`)**: Mount a persistent volume (`local-path` or NFS) at `/root/.ollama` or `/models` so multi-gigabyte model weights (Llama, Mistral, Qwen) are cached across pod restarts instead of re-downloading.
- **Serving Engines**: Deploy **Ollama** (simple local inference) or **vLLM** (high-throughput OpenAI-compatible production API) behind a `ClusterIP` Service + Traefik `Ingress` + Prometheus metrics.
