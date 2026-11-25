# Kubernetes Hands-On Practice Labs (PostgreSQL, Redis & Core Concepts)

This guide is a hands-on lab to practice every remaining core Kubernetes concept on your K3s cluster:
- **`ConfigMap`** & **`Secret`**
- **`Ingress`** (Traefik Layer 7 routing)
- **`PersistentVolumeClaim` (PVC)** & **`StorageClass`** (`local-path` in K3s)
- **`StatefulSet`** & **Headless Service** (for **PostgreSQL** and **Redis**)
- **`HorizontalPodAutoscaler` (HPA)**
- **`Job`** & **`CronJob`**
- **`NetworkPolicy`** & **`RBAC`** (`ServiceAccount`, `Role`, `RoleBinding`)

---

## Lab 0: Create a Practice Namespace

- **What it is**: A virtual cluster boundary inside Kubernetes that groups and isolates resources.
- **Why we use it**: Prevents name collisions (`web-demo` in `lab` won't conflict with `web-demo` in `default`), lets you apply RBAC/quotas per environment, and makes cleanup one command (`kubectl delete namespace lab`).

```bash
kubectl create namespace lab
```

---

## Lab 1: `ConfigMap` + `Ingress` (Upgrade `web-demo`)

- **What `ConfigMap` is**: A key-value object for storing **non-sensitive configuration** (env vars, config files, HTML) separately from container images.
- **Why we use `ConfigMap`**: Keeps container images generic and reusable—you can change `index.html` or `nginx.conf` without rebuilding the Docker image.
- **What `Ingress` is**: A Layer-7 HTTP/HTTPS reverse proxy rule managed by an Ingress Controller (Traefik in K3s).
- **Why we use `Ingress`**: Replaces ugly `NodePort` numbers (`:30080`) with clean domains (`http://web.local`) and paths (`/api`) on standard ports `80`/`443`.

### 1.1 Create `ConfigMap`
```yaml
# lab-web-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-demo-html
  namespace: lab
data:
  index.html: |
    <h1>Hello from ConfigMap + Ingress on K3s!</h1>
```

### 1.2 Deploy Nginx Mounting the `ConfigMap` + `ClusterIP` Service + `Ingress`
```yaml
# lab-web.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-demo
  namespace: lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-demo
  template:
    metadata:
      labels:
        app: web-demo
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 50m
            memory: 32Mi
          limits:
            cpu: 200m
            memory: 128Mi
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html-volume
        configMap:
          name: web-demo-html
---
apiVersion: v1
kind: Service
metadata:
  name: web-demo
  namespace: lab
spec:
  type: ClusterIP
  selector:
    app: web-demo
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-demo-ingress
  namespace: lab
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

### 1.3 Apply & Test
```bash
kubectl apply -f lab-web-configmap.yaml
kubectl apply -f lab-web.yaml

# Test Layer-7 Ingress routing via Traefik (no /etc/hosts edit needed)
curl -H "Host: web.local" http://localhost
```

---

## Lab 2: `StatefulSet` + `Secret` + `PVC` — PostgreSQL

- **What `Secret` is**: A Kubernetes object for storing **sensitive data** (database passwords, API keys, TLS certificates) separately from your manifests.
- **Why we use `Secret`**: Avoids hardcoding passwords in Deployment/StatefulSet YAML; Kubernetes can restrict access to Secrets via RBAC.
- **What `PVC` & `StorageClass` are**: A **`PersistentVolumeClaim` (PVC)** requests durable disk storage from a **`StorageClass`** (`local-path` in K3s).
- **Why we use `PVC`**: Container filesystems are ephemeral—without a PVC, all database tables and rows are lost when a Pod restarts or is rescheduled.
- **What `StatefulSet` & Headless Service (`clusterIP: None`) are**: A workload controller for stateful apps that gives each pod a predictable name (`postgres-0`), stable DNS (`postgres-0.postgres.lab.svc.cluster.local`), ordered startup, and its own dedicated PVC.
- **Why we use `StatefulSet` instead of `Deployment`**: A `Deployment` treats pods as identical/disposable and shares volumes; a `StatefulSet` guarantees each database replica keeps its own identity and its own disk.

### 2.1 Create `Secret` (Credentials) & `ConfigMap` (Init SQL)
```yaml
# lab-postgres-config.yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: lab
type: Opaque
stringData:
  POSTGRES_USER: appuser
  POSTGRES_PASSWORD: SuperSecretPassword123
  POSTGRES_DB: appdb
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-init
  namespace: lab
data:
  init.sql: |
    CREATE TABLE IF NOT EXISTS visits (
      id SERIAL PRIMARY KEY,
      note TEXT NOT NULL,
      created_at TIMESTAMP DEFAULT NOW()
    );
```

### 2.2 Create Headless Service & PostgreSQL `StatefulSet`
```yaml
# lab-postgres.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: lab
spec:
  clusterIP: None   # Headless Service: DNS resolves directly to postgres-0.postgres.lab.svc.cluster.local
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: lab
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16-alpine
        ports:
        - containerPort: 5432
        envFrom:
        - secretRef:
            name: postgres-secret
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "appuser", "-d", "appdb"]
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: pgdata
          mountPath: /var/lib/postgresql/data
        - name: init-script
          mountPath: /docker-entrypoint-initdb.d
      volumes:
      - name: init-script
        configMap:
          name: postgres-init
  volumeClaimTemplates:
  - metadata:
      name: pgdata
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: local-path
      resources:
        requests:
          storage: 1Gi
```

### 2.3 Apply & Test Data Persistence
```bash
kubectl apply -f lab-postgres-config.yaml
kubectl apply -f lab-postgres.yaml

# Watch Pod & PVC creation
kubectl get pods,pvc -n lab -w

# 1. Insert a row into PostgreSQL
kubectl exec -it -n lab postgres-0 -- psql -U appuser -d appdb \
  -c "INSERT INTO visits (note) VALUES ('persisted across pod restart!'); SELECT * FROM visits;"

# 2. Delete the Pod to simulate a crash/reschedule
kubectl delete pod -n lab postgres-0

# 3. Wait for StatefulSet to recreate postgres-0, then verify your data is STILL there!
kubectl exec -it -n lab postgres-0 -- psql -U appuser -d appdb -c "SELECT * FROM visits;"
```

---

## Lab 3: `StatefulSet` + `ConfigMap` + `Secret` + `PVC` — Redis

- **What it is**: Running Redis as a `StatefulSet` with a custom `redis.conf` (`ConfigMap`), password auth (`Secret`), and **AOF (Append-Only File)** persistence backed by a `PVC`.
- **Why we use it**: By default, Redis is purely in-memory—if a pod crashes, all cached/session data disappears. Mounting `appendonly yes` + a `PVC` lets Redis replay its write log on restart so keys survive crashes.

### 3.1 Create `ConfigMap` (`redis.conf`) & `Secret`
```yaml
# lab-redis-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: lab
data:
  redis.conf: |
    appendonly yes
    dir /data
---
apiVersion: v1
kind: Secret
metadata:
  name: redis-secret
  namespace: lab
type: Opaque
stringData:
  REDIS_PASSWORD: RedisSecretPassword123
```

### 3.2 Create Headless Service & Redis `StatefulSet`
```yaml
# lab-redis.yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: lab
spec:
  clusterIP: None
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: lab
spec:
  serviceName: redis
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        command: ["sh", "-c", "redis-server /usr/local/etc/redis/redis.conf --requirepass \"$REDIS_PASSWORD\""]
        ports:
        - containerPort: 6379
        envFrom:
        - secretRef:
            name: redis-secret
        readinessProbe:
          exec:
            command: ["sh", "-c", "redis-cli -a \"$REDIS_PASSWORD\" ping | grep PONG"]
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: redis-data
          mountPath: /data
        - name: redis-conf
          mountPath: /usr/local/etc/redis
      volumes:
      - name: redis-conf
        configMap:
          name: redis-config
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: local-path
      resources:
        requests:
          storage: 512Mi
```

### 3.3 Apply & Test Persistence
```bash
kubectl apply -f lab-redis-config.yaml
kubectl apply -f lab-redis.yaml

# 1. Write a key to Redis
kubectl exec -it -n lab redis-0 -- redis-cli -a RedisSecretPassword123 SET greeting "hello-k3s"

# 2. Delete the Pod
kubectl delete pod -n lab redis-0

# 3. Verify key survived after redis-0 restarts
kubectl exec -it -n lab redis-0 -- redis-cli -a RedisSecretPassword123 GET greeting
```

---

## Lab 4: `HorizontalPodAutoscaler` (HPA) & Vertical Scaling

- **What HPA is**: A Kubernetes controller that automatically scales the **number of Pod replicas** (`2` $\rightarrow$ `6`) based on CPU, memory, or custom metrics. *(Requires `metrics-server` and `resources.requests` on containers.)*
- **What Vertical Scaling (VPA / Manual) is**: Increasing the **CPU and Memory allocated to an existing Pod** (e.g., `256Mi` $\rightarrow$ `2Gi` RAM) instead of adding more Pods.

### Where & How Each Is Used

| Scaling Type | How It Works | Where It Is Used | Why |
|---|---|---|---|
| **Horizontal (HPA)** | Adds/removes Pod **replicas** (`replicas: 2 → 6`) | **Stateless apps** (`web-demo`, APIs, workers) | Identical pods with no local data can immediately share load |
| **Vertical (VPA / Manual)** | Increases **CPU/RAM** on existing Pod(s) | **Stateful apps** (PostgreSQL, Redis, AI models) | Adding `postgres-1` creates an empty PVC and doesn't replicate data automatically |

> **Why not HPA on PostgreSQL / Redis directly?**  
> Scaling a database `StatefulSet` with HPA creates new pods with **empty disks**, risks split-brain writes, and can drop active connections on scale-down. For horizontal read-scaling of databases, use a **Database Operator** (e.g., CloudNativePG, Redis Cluster) that manages replication.

### How Vertical Scaling Is Done
1. **Manual Vertical Scaling**: Update `resources.requests` and `resources.limits` in your Pod spec and re-apply (`kubectl apply -f ...`).
2. **Automatic (`VerticalPodAutoscaler` - VPA)**: Install the VPA add-on and create a `VerticalPodAutoscaler` manifest (`updateMode: "Auto"` or `"Off"` for recommendations only):
   ```yaml
   apiVersion: autoscaling.k8s.io/v1
   kind: VerticalPodAutoscaler
   metadata:
     name: postgres-vpa
     namespace: lab
   spec:
     targetRef:
       apiVersion: apps/v1
       kind: StatefulSet
       name: postgres
     updatePolicy:
       updateMode: "Off"   # "Off" = recommend CPU/RAM safely; "Auto" = restart pod with new sizes
   ```

### Practice: Apply HPA to `web-demo` (Stateless)

```yaml
# lab-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-demo-hpa
  namespace: lab
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-demo
  minReplicas: 2
  maxReplicas: 6
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

```bash
kubectl apply -f lab-hpa.yaml
kubectl get hpa -n lab

# Optional: generate HTTP load to watch replicas scale up
kubectl run -n lab load-gen --rm -it --restart=Never --image=busybox -- \
  sh -c "while true; do wget -q -O- http://web-demo; done"
```

---

## Lab 5: `Job` & `CronJob`

- **What `Job` is**: Creates one or more Pods to run a task to **completion** (exit code `0`) and then stops (unlike a `Deployment`, which restarts containers forever).
- **Why we use `Job`**: Ideal for finite tasks like database schema migrations, data seeding, or batch processing.
- **What `CronJob` is**: Schedules `Job`s to run automatically on a recurring cron schedule (`*/5 * * * *`).
- **Why we use `CronJob`**: Automates periodic maintenance like database backups, report generation, or health checks.

```yaml
# lab-jobs.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pg-check-job
  namespace: lab
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: pg-check
        image: postgres:16-alpine
        envFrom:
        - secretRef:
            name: postgres-secret
        command: ["sh", "-c", "PGPASSWORD=$POSTGRES_PASSWORD psql -h postgres -U $POSTGRES_USER -d $POSTGRES_DB -c 'SELECT count(*) FROM visits;'"]
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: redis-ping-cron
  namespace: lab
spec:
  schedule: "*/5 * * * *"   # Every 5 minutes
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: redis-ping
            image: redis:7-alpine
            envFrom:
            - secretRef:
                name: redis-secret
            command: ["sh", "-c", "redis-cli -h redis -a $REDIS_PASSWORD ping"]
```

```bash
kubectl apply -f lab-jobs.yaml
kubectl get jobs,cronjobs -n lab
kubectl logs -n lab job/pg-check-job
```

---

## Lab 6: Security — `NetworkPolicy` & `RBAC`

### 6.1 `NetworkPolicy` (Pod-Level Firewall)
- **What it is**: A Kubernetes firewall specification that controls which Pods or namespaces are allowed to send/receive network traffic to other Pods.
- **Why we use it**: By default in Kubernetes, **every Pod can talk to every other Pod** across the cluster. `NetworkPolicy` enforces zero-trust isolation so only `web-demo` can reach `postgres` (`5432`) and `redis` (`6379`).

Lock down `postgres` and `redis` so **only** pods with label `app: web-demo` can connect to ports `5432` and `6379`:

```yaml
# lab-netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-db
  namespace: lab
spec:
  podSelector:
    matchExpressions:
    - key: app
      operator: In
      values: ["postgres", "redis"]
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web-demo
    ports:
    - protocol: TCP
      port: 5432
    - protocol: TCP
      port: 6379
```

```bash
kubectl apply -f lab-netpol.yaml

# Test: a random pod WITHOUT app=web-demo should be blocked from reaching postgres:5432
kubectl run -n lab tester --rm -it --restart=Never --image=busybox -- nc -zv -w 3 postgres 5432
```

### 6.2 `RBAC` (`ServiceAccount` + `Role` + `RoleBinding`)
- **What it is**: **Role-Based Access Control**—`ServiceAccount` (identity for a Pod/app), `Role` (permissions within a namespace), and `RoleBinding` (attaches the Role to the ServiceAccount).
- **Why we use it**: Enforces **least privilege** so apps or users only have the exact Kubernetes API permissions they need (e.g., read Pods in `lab`, cannot delete Pods or read Secrets in other namespaces).

Grant a `ServiceAccount` read-only access to Pods in the `lab` namespace only:

```yaml
# lab-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-reader-sa
  namespace: lab
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: lab
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: lab
subjects:
- kind: ServiceAccount
  name: pod-reader-sa
  namespace: lab
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f lab-rbac.yaml

# Verify permissions using kubectl auth can-i
kubectl auth can-i list pods -n lab --as=system:serviceaccount:lab:pod-reader-sa      # yes
kubectl auth can-i delete pods -n lab --as=system:serviceaccount:lab:pod-reader-sa    # no
kubectl auth can-i list pods -n default --as=system:serviceaccount:lab:pod-reader-sa  # no
```

---

## Cleanup When Finished

```bash
# Deletes every resource and PVC created in the lab namespace
kubectl delete namespace lab
```
