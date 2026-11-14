# K3s Production Setup & Runbook

A guide and runbook for operating production-grade stateful, stateless, and AI model workloads on K3s.

---

## 🎯 Goals

- Host **stateful applications** at production level.
- Host **stateless applications** at production level.
- Host **AI models & inferencing workloads** at production level.

---

## 🐧 Linux & System Operations

Detailed system logging and monitoring guides:
- 📖 [Linux `systemctl` & `journalctl` Cheat Sheet and Guide](docs/linux.md)

### Key Inspection Commands
```bash
# Check k3s eviction thresholds and startup config
journalctl -u k3s -b | grep -A 10 "evictionHard:"

# Stream recent k3s logs
journalctl -u k3s -n 50 --no-pager
```

---

## 🐳 Docker & Docker Compose

### Common Docker Commands
```bash
# List active containers
docker ps

# Check disk usage by containers, images, and volumes
docker system df

# Clean unused containers, networks, and dangling images
docker system prune

# Deep clean including unused images and volumes
docker system prune -a --volumes
```

### Docker Compose
```bash
# Start services
docker compose -f <filename.yaml> up -d

# Stop and remove containers
docker compose -f <filename.yaml> down
```

---

## ☸️ K3s & Kubernetes Operations

### Common Commands
```bash
# Check cluster nodes
kubectl get nodes -o wide

# Check pods across all namespaces or wide output
kubectl get pods -o wide
kubectl get svc
kubectl get all

# Inspect specific resources
kubectl describe pod <pod-name>
kubectl describe node <node-name>
kubectl describe service <service-name>
```

### Exposing Services & Traffic Access

Kubernetes pods are ephemeral and their private IPs change dynamically. Services provide stable endpoints and load balancing across pods:

```text
[ External / Browser ]
           │
           ▼
[ NodePort: <Node-IP>:30080 ]
           │
           ▼
[ ClusterIP: 10.43.65.214:80 ] ──► Load Balances ──► [ Pod 1: 10.42.0.10 ]
                                                  ──► [ Pod 2: 10.42.0.12 ]
                                                  ──► [ Pod 3: 10.42.0.14 ]
```

#### 1. `ClusterIP` (Default Internal Service)
- **What it is**: An internal virtual IP allocated from the cluster's internal subnet (`10.43.0.0/16`).
- **Scope**: Internal to cluster and host machine only.
- **Use Case**: East-west traffic between microservices (e.g. backend API talking to database or cache). Not reachable from outside the cluster.
- **Example Access**: `curl http://10.43.65.214:80` or via internal DNS `http://web-demo.default.svc.cluster.local`.

#### 2. `NodePort` (Exposing on the Host Network)
- **What it is**: Builds directly on top of `ClusterIP` by allocating a dedicated high port (**30000–32767**) on every node in the cluster.
- **Scope**: Cluster + Local Area Network (LAN) / External.
- **Use Case**: Exposing applications on bare-metal or homelab clusters without needing an external cloud load balancer. Accessible from your browser or any device on the same local network.
- **Example Access**: `curl http://localhost:30080` or `curl http://192.168.1.7:30080`.

#### 3. `Port Forward` (`kubectl port-forward`)
- **What it is**: A client-side debugging utility that creates an encrypted TLS tunnel from your local machine to a pod or service via the Kubernetes API server.
- **Scope**: Local machine only while the terminal command is running.
- **Use Case**: Ad-hoc debugging, testing, or inspecting admin dashboards and databases without exposing them to the network.
- **Example Command**:
  ```bash
  # Forward local port 8080 to service web-demo port 80
  kubectl port-forward svc/web-demo 8080:80
  # Now access at http://localhost:8080
  ```

#### Comparison Summary

| Feature | `ClusterIP` | `NodePort` | `kubectl port-forward` |
|---|---|---|---|
| **Primary Purpose** | Internal communication | Exposing service to host/LAN | Ad-hoc debugging & inspection |
| **Accessibility** | Inside cluster only | Node IP + Port on host/LAN | Local workstation only |
| **Port Range** | Any port (`80`, `443`, `5432`) | `30000–32767` | Any free local port (`8080`, `3000`) |
| **Requires Service?** | Yes | Yes (creates ClusterIP underneath) | No (can target Pod or Service directly) |
| **Persistence** | Permanent | Permanent | Temporary (active only while running) |
| **Typical Use Case** | Databases, internal APIs | Homelab/LAN access, staging | Temporary troubleshooting |

### Health Checks: Liveness & Readiness Probes

Probes are configured in [`nginx/nginx-deployment.yaml`](nginx/nginx-deployment.yaml) so Kubernetes can manage traffic routing and self-heal unhealthy containers:

- **`readinessProbe`** (`initialDelaySeconds: 3`, `periodSeconds: 5`): Checks `GET /` on port `80`. If it fails, Kubernetes removes the pod from the Service endpoints so traffic is not routed to it until it is ready.
- **`livenessProbe`** (`initialDelaySeconds: 10`, `periodSeconds: 10`): Checks `GET /` on port `80`. If it fails repeatedly, Kubernetes restarts the container automatically.
- **Resource Limits**: Requests `50m` CPU / `32Mi` memory and limits `200m` CPU / `128Mi` memory to prevent resource starvation or OOM issues.

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 3
  periodSeconds: 5
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 10
  periodSeconds: 10
```

#### Apply & Verify Rollout
```bash
# Apply updated deployment
kubectl apply -f nginx/nginx-deployment.yaml

# Watch zero-downtime rolling update
kubectl rollout status deployment/web-demo

# Verify probe configuration on running pods
kubectl describe pod -l app=web-demo | grep -E "(Liveness|Readiness)"
```

---

## 📊 Monitoring & Alerting (`kube-prometheus-stack`)

Cluster and application observability is powered by **Prometheus**, **Alertmanager**, and **Grafana** deployed via Helm into the `monitoring` namespace.

### 1. Install `kube-prometheus-stack` (K3s)
Using GitHub's OCI registry avoids `release-assets.githubusercontent.com` download timeouts, and disabling separate control-plane scrapers prevents false target-down alerts on single-binary K3s:

```bash
kubectl create namespace monitoring

helm install prometheus oci://ghcr.io/prometheus-community/charts/kube-prometheus-stack \
  --namespace monitoring \
  --set kubeControllerManager.enabled=false \
  --set kubeScheduler.enabled=false \
  --set kubeEtcd.enabled=false \
  --set kubeProxy.enabled=false

# Verify all monitoring pods are Running
kubectl get pods -n monitoring -w
```

### 2. Access Dashboards & UIs

#### Grafana (`http://localhost:3000`)
```bash
# Get the default 'admin' password
kubectl --namespace monitoring get secrets prometheus-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d ; echo

# Forward local port 3000 to Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
```
*In Grafana, open **Dashboards → Kubernetes / Compute Resources / Namespace (Workloads)** and select namespace **`default`** to inspect `web-demo`.*

#### Prometheus (`http://localhost:9090`) & Alertmanager (`http://localhost:9093`)
```bash
# Prometheus UI (Targets, PromQL, Alerts)
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090

# Alertmanager UI (Fired & Grouped Alerts)
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-alertmanager 9093:9093
```

### 3. Custom Alert Rules (`web-demo`)
Custom alerts are defined in [`nginx/web-demo-alerts.yaml`](nginx/web-demo-alerts.yaml) as a `PrometheusRule` CRD:
- **`WebDemoReplicasMismatch`**: Fires if available replicas drop below desired replicas for `1m`.
- **`WebDemoPodRestarting`**: Fires if any `web-demo` pod restarts in the last `10m`.
- **`WebDemoHighMemoryUsage`**: Fires if `nginx` memory exceeds `80%` of its `128Mi` limit for `2m`.

> **Note:** `metadata.labels.release: prometheus` is required so the Prometheus Operator `ruleSelector` discovers the rule.

```bash
# Apply and verify custom rules
kubectl apply -f nginx/web-demo-alerts.yaml
kubectl get prometheusrules -n monitoring web-demo-alerts
```

#### Testing an Alert
```bash
# Trigger WebDemoReplicasMismatch by breaking the image temporarily
kubectl set image deployment/web-demo nginx=nginx:invalid-tag

# Watch Inactive -> Pending -> Firing at http://localhost:9090/alerts
# Restore deployment when done
kubectl apply -f nginx/nginx-deployment.yaml
```

### 4. Alert Notifications (`AlertmanagerConfig`)
Alert routing and notifications are configured via the `AlertmanagerConfig` CRD in [`nginx/alertmanager-webhook.yaml`](nginx/alertmanager-webhook.yaml):

```text
[ Prometheus ] ──(Alert Fires)──► [ Alertmanager ] ──► [ Webhook / Slack / Email ]
```

- **Webhook Testing**: Uses `webhookConfigs` with `sendResolved: true` to deliver both `firing` and `resolved` JSON payloads (e.g., to `webhook.site`).
- **Production Channels**: Swap `webhookConfigs` for `slackConfigs`, `discordConfigs`, or `emailConfigs` and store credentials/URLs in a Kubernetes `Secret`.

```bash
# Apply and verify AlertmanagerConfig
kubectl apply -f nginx/alertmanager-webhook.yaml
kubectl get alertmanagerconfig -n default
```

---

## 🛠️ Troubleshooting & Issue Log

### Issue: Pod Stuck in `Pending` (Disk Pressure Taint)

#### Symptom
Pods remain in `Pending` state indefinitely. Running `kubectl describe pod <pod-name>` reports:
```text
Warning  FailedScheduling  default-scheduler  0/1 nodes are available: 1 node(s) had untolerated taint(s).
```

Checking the node reveals:
```bash
kubectl describe node <node-name> | grep -E "Taints|DiskPressure"
```
```text
Taints:          node.kubernetes.io/disk-pressure:NoSchedule
DiskPressure     True    KubeletHasDiskPressure    kubelet has disk pressure
```

#### Root Cause
Kubelet marks the node with `DiskPressure=True` and applies `node.kubernetes.io/disk-pressure:NoSchedule` when free disk space falls below the hard eviction threshold.

Even after freeing up disk space, the condition may persist due to **`evictionMinimumReclaim`**:
```bash
journalctl -u k3s -b | grep -A 10 "evictionHard:"
```
```text
evictionHard:
  imagefs.available: 5%
  nodefs.available: 5%
evictionMinimumReclaim:
  imagefs.available: 10%
  nodefs.available: 10%
evictionPressureTransitionPeriod: 5m0s
```

* **Clear Condition Formula**: Free space must rise above **Threshold (5%) + Min Reclaim (10%) = 15% free space** ($\le 85\%$ used).
* **Transition Hold**: Kubelet holds the condition for up to **5 minutes** (`evictionPressureTransitionPeriod`) to avoid flapping.

#### Mitigation & Fix
1. **Free Disk Space**:
   ```bash
   # Reclaim docker space
   docker system prune -a --volumes

   # Clear cache and package manager artifacts
   rm -rf ~/.cache/*
   sudo apt clean
   sudo journalctl --vacuum-size=500M
   ```

2. **Verify Disk Status**:
   ```bash
   df -h /
   ```
   Ensure root filesystem usage is $\le 85\%$ ($> 15\%$ available).

3. **Reset State (Optional)**:
   To immediately clear the 5-minute transition period once space is reclaimed:
   ```bash
   sudo systemctl restart k3s
   ```

4. **Verify Pod Scheduling**:
   ```bash
   kubectl get pods -o wide
   ```
