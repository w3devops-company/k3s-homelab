# Kubernetes (`kubectl`) Cheatsheet & Production Debugging Guide

A quick reference for daily Kubernetes operations and real-world production troubleshooting.

---

## 1. Contexts, Clusters & Namespaces

```bash
kubectl config get-contexts                             # List all contexts
kubectl config current-context                          # Show active context
kubectl config use-context <context-name>               # Switch context
kubectl config set-context --current --namespace=<ns>   # Set default namespace

kubectl cluster-info                                    # Cluster control plane info
kubectl get ns                                          # List namespaces
kubectl create ns <ns>                                  # Create namespace
```

---

## 2. Viewing & Inspecting Resources

```bash
kubectl get all -n <ns>                                 # Pods, services, deployments, replicasets
kubectl get pods -A                                     # Pods across all namespaces
kubectl get pods -o wide                                # Show node, IP, and status
kubectl get pods -w                                     # Watch pod status changes live
kubectl get events -A --sort-by=.metadata.creationTimestamp  # Recent cluster events

kubectl describe pod <pod> -n <ns>                      # Detailed state, conditions & events
kubectl get pod <pod> -n <ns> -o yaml                   # Full resource manifest
kubectl api-resources                                   # List all supported API resource types
```

---

## 3. Pods, Logs & Interactive Debugging

```bash
kubectl logs <pod> -n <ns>                              # Container logs
kubectl logs <pod> -c <container> -n <ns>               # Logs for a specific container
kubectl logs <pod> -f --tail=100 -n <ns>                # Follow last 100 lines
kubectl logs <pod> --previous -n <ns>                   # Logs from previous crashed container
kubectl logs -l app=<label> --all-containers -n <ns>    # Logs across pods matching a label

kubectl exec -it <pod> -n <ns> -- /bin/sh               # Interactive shell in pod
kubectl exec -it <pod> -c <container> -- /bin/bash      # Shell in specific container
kubectl port-forward svc/<svc> 8080:80 -n <ns>          # Forward local 8080 -> service 80
kubectl cp <ns>/<pod>:/path/to/file ./local-file        # Copy file from pod to local
```

---

## 4. Deployments, Rollouts & Scaling

```bash
kubectl apply -f manifest.yaml                          # Apply/update resources
kubectl delete -f manifest.yaml                         # Delete resources

kubectl scale deploy/<name> --replicas=3 -n <ns>        # Scale replicas
kubectl set image deploy/<name> <container>=<img:tag>   # Update container image

kubectl rollout status deploy/<name> -n <ns>            # Watch rollout progress
kubectl rollout history deploy/<name> -n <ns>           # View revision history
kubectl rollout undo deploy/<name> -n <ns>              # Roll back to previous revision
kubectl rollout undo deploy/<name> --to-revision=2      # Roll back to specific revision
kubectl rollout restart deploy/<name> -n <ns>           # Trigger rolling restart
```

---

## 5. Services, Ingress & Networking

```bash
kubectl get svc,ep,ing -n <ns>                          # Services, endpoints, and ingresses
kubectl describe svc <svc> -n <ns>                      # Check selectors & target ports
kubectl get endpoints <svc> -n <ns>                     # Verify backend pod IPs exist
kubectl get netpol -A                                   # List NetworkPolicies

# Quick temporary network debug pod
kubectl run netdebug --rm -it --image=nicolaka/netshoot -- /bin/bash
```

---

## 6. ConfigMaps, Secrets & Storage

```bash
kubectl get cm,secret -n <ns>                           # List ConfigMaps and Secrets
kubectl get secret <name> -n <ns> -o jsonpath='{.data.<key>}' | base64 -d  # Decode secret key

kubectl get pv,pvc,sc -A                                # PersistentVolumes, Claims, StorageClasses
kubectl describe pvc <pvc> -n <ns>                      # Debug unbound/pending PVC
```

---

## 7. Nodes, Scheduling & Maintenance

```bash
kubectl get nodes -o wide                               # Node status, IPs, kernel & runtime
kubectl describe node <node>                            # Conditions, taints, allocated resources
kubectl top nodes                                       # Node CPU & memory usage (needs metrics-server)
kubectl top pods -A --sort-by=memory                    # Top memory-consuming pods
kubectl top pods -A --sort-by=cpu                       # Top CPU-consuming pods

kubectl cordon <node>                                   # Mark node unschedulable
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data  # Safely evict pods for maintenance
kubectl uncordon <node>                                 # Re-enable scheduling on node
kubectl taint nodes <node> key=value:NoSchedule         # Add taint
kubectl taint nodes <node> key=value:NoSchedule-        # Remove taint
```

---

## 8. Useful Filtering & Output Tricks

```bash
# Find all non-running / unhealthy pods across the cluster
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded

# Sort pods by restart count
kubectl get pods -A --sort-by='.status.containerStatuses[0].restartCount'

# List all container images running in a namespace
kubectl get pods -n <ns> -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'

# Dry-run and diff before applying to production
kubectl diff -f manifest.yaml
kubectl apply -f manifest.yaml --dry-run=server
```

---

## 9. Production Debugging Scenarios

### Scenario 1: `CrashLoopBackOff` / Container Keeps Restarting
**Symptoms:** Pod restarts repeatedly; `STATUS` shows `CrashLoopBackOff` or `Error`.
```bash
kubectl describe pod <pod> -n <ns>                      # Check Last State, Exit Code & Reason
kubectl logs <pod> --previous -n <ns>                   # Logs from the crashed run
kubectl logs <pod> -c <init-container> -n <ns>          # Check if an initContainer failed
```
**Common Exit Codes:**
- `0` — Process exited cleanly (container command finished; check `command`/`args`)
- `1` — Application error / unhandled exception / bad config
- `137` — `SIGKILL` (usually **OOMKilled** or forced termination)
- `139` — `SIGSEGV` (segmentation fault)
- `143` — `SIGTERM` (graceful shutdown signal received)

---

### Scenario 2: `OOMKilled` (Exit Code 137)
**Symptoms:** Pod restarts under load; `describe pod` shows `Reason: OOMKilled`.
```bash
kubectl describe pod <pod> -n <ns> | grep -A 10 "Last State"
kubectl top pod <pod> -n <ns> --containers              # Live memory usage per container
kubectl get pod <pod> -n <ns> -o jsonpath='{.spec.containers[*].resources}'  # Check limits/requests
```
**Fix:** Increase `resources.limits.memory` (or fix memory leak / JVM/Node heap flags like `-Xmx` or `--max-old-space-size` so the runtime stays below the container limit).

---

### Scenario 3: Pod Stuck in `Pending`
**Symptoms:** Pod is never scheduled onto a node.
```bash
kubectl describe pod <pod> -n <ns>                      # Look at Events at the bottom
kubectl get nodes                                       # Check Ready status & scheduling
kubectl describe nodes | grep -A 5 "Allocated resources" # Check CPU/memory exhaustion
kubectl get pvc -n <ns>                                 # Check if a required PVC is Pending
```
**Common Causes:** Insufficient CPU/memory on nodes, unmatched `nodeSelector`/affinity, missing taint toleration, or unbound PVC.

---

### Scenario 4: `ImagePullBackOff` / `ErrImagePull`
**Symptoms:** Pod cannot download container image.
```bash
kubectl describe pod <pod> -n <ns>                      # Check exact registry error in Events
kubectl get pod <pod> -n <ns> -o jsonpath='{.spec.imagePullSecrets}'  # Verify pull secret
kubectl get secret <pull-secret> -n <ns>                # Confirm secret exists in same namespace
```
**Common Causes:** Typo in image name/tag, expired registry credentials, missing `imagePullSecrets`, or outbound firewall/proxy blocking the registry.

---

### Scenario 5: Service or Ingress Unreachable (502 / 503 / Timeout)
**Symptoms:** App is running inside the pod, but traffic fails to reach it.
```bash
# 1. Check if the Service actually has backend pod IPs
kubectl get endpoints <svc> -n <ns>
# Empty endpoints? Service selector labels do NOT match Pod labels, or readinessProbe is failing.

# 2. Verify selector & port mapping (port vs targetPort)
kubectl describe svc <svc> -n <ns>
kubectl get pods -n <ns> --show-labels

# 3. Test from inside the cluster (bypass Ingress)
kubectl run curl-test --rm -it --image=curlimages/curl -n <ns> -- curl -v http://<svc>.<ns>.svc.cluster.local:<port>

# 4. Test DNS resolution (CoreDNS)
kubectl run dns-test --rm -it --image=busybox:1.36 -n <ns> -- nslookup <svc>.<ns>.svc.cluster.local
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# 5. Check Ingress controller logs & NetworkPolicies
kubectl describe ing <ingress> -n <ns>
kubectl get netpol -n <ns>
```

---

### Scenario 6: Node `NotReady` or Resource Pressure
**Symptoms:** Pods evicted or unable to schedule; node shows `NotReady`, `DiskPressure`, or `MemoryPressure`.
```bash
kubectl describe node <node>                            # Check Conditions section
kubectl get events -A --field-selector involvedObject.name=<node>
```
**On the affected node (via SSH):**
```bash
systemctl status kubelet                                # Or `systemctl status k3s` / `k3s-agent`
journalctl -u kubelet -n 100 --no-pager                 # Kubelet error logs
df -h                                                   # Check disk space (/var/lib/containerd, /var/log)
free -h                                                 # Check host RAM
crictl ps -a                                            # Inspect container runtime directly
```

---

### Scenario 7: Pod or Namespace Stuck in `Terminating`
**Symptoms:** Resource won't delete after `kubectl delete`.
```bash
# Force-delete a stuck pod (use cautiously with StatefulSets to avoid split-brain)
kubectl delete pod <pod> -n <ns> --grace-period=0 --force

# Check what finalizers or resources are blocking a Terminating namespace
kubectl get ns <ns> -o jsonpath='{.spec.finalizers}'
kubectl api-resources --verbs=list --namespaced -o name | xargs -n 1 kubectl get --show-kind --ignore-not-found -n <ns>
```

---

### Scenario 8: Bad Production Rollout / Immediate Rollback
**Symptoms:** Error rate spikes right after a deployment.
```bash
kubectl rollout status deploy/<name> -n <ns>
kubectl rollout history deploy/<name> -n <ns>
kubectl rollout undo deploy/<name> -n <ns>              # Immediate rollback to previous revision
kubectl get rs -n <ns>                                  # Verify old ReplicaSet scales back up
```

---

### Scenario 9: Debugging Minimal / Distroless Containers (No `sh` or `curl`)
**Symptoms:** `kubectl exec` fails because the image has no shell or debugging binaries.
```bash
# Attach an ephemeral debug container sharing the target container's process namespace
kubectl debug -it <pod> -n <ns> --image=nicolaka/netshoot --target=<container>

# Debug a node's host filesystem & processes directly via a privileged debug pod
kubectl debug node/<node> -it --image=ubuntu
```

---

### Scenario 10: K3s-Specific Quick Checks
If running **K3s** on Ubuntu:
```bash
sudo systemctl status k3s                               # Control-plane service status
sudo systemctl status k3s-agent                         # Worker node service status
sudo journalctl -u k3s -n 100 --no-pager                # K3s server logs
sudo k3s kubectl get nodes                              # Built-in kubectl
sudo k3s crictl ps                                      # Inspect containerd containers
sudo k3s check-config                                   # Verify OS kernel/cgroup requirements
```
**Key K3s Paths:**
- Kubeconfig: `/etc/rancher/k3s/k3s.yaml`
- Config file: `/etc/rancher/k3s/config.yaml`
- Auto-deployed manifests: `/var/lib/rancher/k3s/server/manifests/`
- Data & embedded DB: `/var/lib/rancher/k3s/server/db/`
