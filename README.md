# nids-iyp
How to create your own instance of Internet Yellow Pages (IYP) and run it on an NRP node
# IYP on NRP Nautilus — Complete Setup Guide

## Prerequisites

- `kubectl` configured with access to the `esculate-caida` namespace
- AWS S3 credentials for the `iyp-caida` bucket on `s3-west.nrp-nautilus.io`

---

## Step 1 — Create the PersistentVolumeClaim

Save as `pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: iyp-storage
  namespace: esculate-caida
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 200Gi
  storageClassName: rook-cephfs
```

Apply:

```bash
kubectl apply -f pvc.yaml -n esculate-caida
kubectl get pvc -n esculate-caida
# Wait until STATUS shows "Bound"
```

---

## Step 2 — Download the Dump File

Save as `iyp-download-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: iyp-download
  namespace: esculate-caida
spec:
  containers:
  - name: downloader
    image: ubuntu
    command: ["sleep", "7200"]
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "1Gi"
        cpu: "1000m"
    volumeMounts:
    - name: iyp-data
      mountPath: /data
  volumes:
  - name: iyp-data
    persistentVolumeClaim:
      claimName: iyp-storage
```

Apply and exec in:

```bash
kubectl apply -f iyp-download-pod.yaml -n esculate-caida

# Wait for Running
kubectl get pod iyp-download -n esculate-caida -w

kubectl exec -it iyp-download -n esculate-caida -- bash
```

Inside the pod:

```bash
# Verify PVC is mounted correctly (should show ~200GB filesystem)
df -h /data

# Install awscli
apt-get update && apt-get install -y --no-install-recommends python3-pip
pip3 install awscli

# Configure S3 credentials
aws configure
# Enter your access key and secret key, leave region and format blank

# Test bucket access
aws s3 ls s3://iyp-caida/ --endpoint-url https://s3-west.nrp-nautilus.io
# Should show: PRE iyp/

# Download the dump to the PVC
mkdir -p /data/dumps
aws s3 cp s3://iyp-caida/iyp/iyp-2024-07-22.dump /data/dumps/neo4j.dump \
  --endpoint-url https://s3-west.nrp-nautilus.io

# Verify download completed (~4.1GB)
ls -lh /data/dumps/neo4j.dump
```

Exit and delete the download pod:

```bash
exit
kubectl delete pod iyp-download -n esculate-caida
```

---

## Step 3 — Load the Dump and Run Neo4j

Save as `iyp-neo4j-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: iyp-neo4j
  namespace: esculate-caida
  labels:
    k8s-app: iyp-neo4j
spec:
  initContainers:
  - name: load-dump
    image: neo4j/neo4j-admin:2025-community-debian
    command:
    - bash
    - -c
    - |
      if [ -d /data/databases/neo4j ]; then
        echo "Database already loaded, skipping."
      else
        echo "Loading dump..."
        mkdir -p /data/databases /data/transactions
        chown -R neo4j:neo4j /data/databases /data/transactions /data/dumps
        su -s /bin/bash neo4j -c "neo4j-admin database load neo4j \
          --from-path=/data/dumps \
          --overwrite-destination=true \
          --verbose"
        echo "Done."
      fi
    resources:
      requests:
        memory: "8Gi"
        cpu: "1000m"
      limits:
        memory: "16Gi"
        cpu: "2000m"
    volumeMounts:
    - name: iyp-data
      mountPath: /data
  containers:
  - name: neo4j
    image: neo4j:2025-community
    ports:
    - containerPort: 7474
    - containerPort: 7687
    env:
    - name: NEO4J_AUTH
      value: "neo4j/password"
    - name: NEO4J_server_directories_data
      value: /data
    - name: NEO4J_server_config_strict__validation_enabled
      value: "false"
    - name: NEO4J_dbms_security_procedures_unrestricted
      value: "gds.*,apoc.*"
    - name: NEO4J_dbms_connector_bolt_tls__level
      value: "DISABLED"
    - name: NEO4J_dbms_connector_bolt_listen__address
      value: "0.0.0.0:7687"
    - name: NEO4J_dbms_connector_http_listen__address
      value: "0.0.0.0:7474"
    - name: NEO4J_db_recovery_fail__on__missing__files
      value: "false"
    resources:
      requests:
        memory: "6Gi"
        cpu: "1000m"
      limits:
        memory: "8Gi"
        cpu: "2000m"
    volumeMounts:
    - name: iyp-data
      mountPath: /data
  volumes:
  - name: iyp-data
    persistentVolumeClaim:
      claimName: iyp-storage
```

Apply and watch the logs:

```bash
kubectl apply -f iyp-neo4j-pod.yaml -n esculate-caida

# Watch init container load the dump (~10-15 minutes)
kubectl logs iyp-neo4j -n esculate-caida -c load-dump -f
# Wait for: "Done: 88 files, 40.51GiB processed"

# Then watch Neo4j start (~5 minutes)
kubectl logs iyp-neo4j -n esculate-caida -c neo4j -f
# Wait for: "Started."
```

> **Note:** On subsequent restarts the init container will print `Database already loaded, skipping.` and Neo4j will start immediately.

---

## Step 4 — Verify Neo4j is Working

Port-forward in a second terminal:

```bash
kubectl port-forward pod/iyp-neo4j 7474:7474 7687:7687 -n esculate-caida
```

Open http://localhost:7474 and login:

- **Username:** `neo4j`
- **Password:** `password`
- **Connect URL:** `bolt://localhost:7687`

Run a test query:

```cypher
MATCH (n) RETURN distinct labels(n), count(n) ORDER BY count(n) DESC
```

Should return a table of node types with millions of total nodes.

---

## Step 5 — Expose Neo4j via Ingress

Save as `iyp-neo4j-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: iyp-neo4j-svc
  namespace: esculate-caida
spec:
  selector:
    k8s-app: iyp-neo4j
  ports:
  - name: http
    port: 7474
    targetPort: 7474
  - name: bolt
    port: 7687
    targetPort: 7687
  type: ClusterIP
```

Save as `iyp-neo4j-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: iyp-neo4j-ingress
  namespace: esculate-caida
spec:
  ingressClassName: haproxy
  rules:
  - host: iyp-caida.nrp-nautilus.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: iyp-neo4j-svc
            port:
              number: 7474
  tls:
  - hosts:
    - iyp-caida.nrp-nautilus.io
```

Apply:

```bash
kubectl apply -f iyp-neo4j-service.yaml -n esculate-caida
kubectl apply -f iyp-neo4j-ingress.yaml -n esculate-caida
```

Neo4j browser UI will be available at https://iyp-caida.nrp-nautilus.io.

> **Note:** Bolt (port 7687) cannot be exposed via NRP Ingress. Use `kubectl port-forward` for direct database connections.

---

## Step 6 — Public Dump File Server (Optional)

This serves the raw dump file for others to download at https://iyp-files.nrp-nautilus.io.

> **Important:** The PVC is ReadWriteOnce — you cannot run `iyp-fileserver` and `iyp-neo4j` simultaneously. Delete one before starting the other.

Save as `iyp-nginx-config.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: iyp-nginx-config
  namespace: esculate-caida
data:
  default.conf: |
    server {
      listen 80;
      location / {
        root /usr/share/nginx/html;
        autoindex on;
        autoindex_exact_size off;
        autoindex_localtime on;
      }
    }
```

Save as `iyp-fileserver-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: iyp-fileserver
  namespace: esculate-caida
  labels:
    k8s-app: iyp-fileserver
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
    volumeMounts:
    - name: iyp-data
      mountPath: /usr/share/nginx/html
      subPath: dumps
    - name: nginx-config
      mountPath: /etc/nginx/conf.d
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "250m"
  volumes:
  - name: iyp-data
    persistentVolumeClaim:
      claimName: iyp-storage
  - name: nginx-config
    configMap:
      name: iyp-nginx-config
```

Save as `iyp-fileserver-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: iyp-fileserver-svc
  namespace: esculate-caida
spec:
  selector:
    k8s-app: iyp-fileserver
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  type: ClusterIP
```

Save as `iyp-fileserver-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: iyp-fileserver-ingress
  namespace: esculate-caida
  annotations:
    haproxy.org/response-set-header: |
      Content-Disposition "attachment"
spec:
  ingressClassName: haproxy
  rules:
  - host: iyp-files.nrp-nautilus.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: iyp-fileserver-svc
            port:
              number: 80
  tls:
  - hosts:
    - iyp-files.nrp-nautilus.io
```

Apply:

```bash
kubectl delete pod iyp-neo4j -n esculate-caida
kubectl apply -f iyp-nginx-config.yaml -n esculate-caida
kubectl apply -f iyp-fileserver-pod.yaml -n esculate-caida
kubectl apply -f iyp-fileserver-service.yaml -n esculate-caida
kubectl apply -f iyp-fileserver-ingress.yaml -n esculate-caida
```

---

## Restarting After a Pod Deadline Kill

Since NRP kills pods after ~6 hours, here's how to restart Neo4j quickly:

```bash
# Check current state
kubectl get pods -n esculate-caida

# Reapply the pod (init container will skip the load)
kubectl apply -f iyp-neo4j-pod.yaml -n esculate-caida

# Watch startup (~5 minutes)
kubectl logs iyp-neo4j -n esculate-caida -c neo4j -f

# Port-forward once Started
kubectl port-forward pod/iyp-neo4j 7474:7474 7687:7687 -n esculate-caida
```

> **Note:** After a restart, wait for the rate limit to clear before logging in if you see auth errors. Wait 5 minutes and try again.

---

## Important Notes & Lessons Learned

| Issue | Solution |
|---|---|
| PVC is ReadWriteOnce | Only one pod can mount at a time — delete current pod before starting a new one |
| Pod deadline ~6 hours | Pods get killed but PVC data persists — just reapply the YAML |
| apt packages lost on restart | Reinstall each time you exec into a fresh pod |
| neo4j-admin OOM on load | Needs 8GB+ free RAM on node — request `memory: 16Gi` limit |
| Version must match | Use `neo4j:2025-community` to match `neo4j/neo4j-admin:2025-community-debian` |
| Neo4j data directory | Set `NEO4J_server_directories_data=/data` — Neo4j appends `databases/` internally |
| Transaction logs missing | Add `NEO4J_db_recovery_fail__on__missing__files=false` — safe after a dump load |
| Load must run as neo4j user | Use `su -s /bin/bash neo4j -c "..."` — running as root causes permission errors |
| Auth rate limit | If locked out, wait 5 minutes before retrying login |
| S3 endpoint | Use `s3-west.nrp-nautilus.io` |
| awscli not in apt | Install via `pip3 install awscli` |
| Mac SSL issues | Use curl/aws inside the pod, not from Mac |
| Bolt not exposable via Ingress | NRP only supports HTTP ingress — use `kubectl port-forward` for Bolt |
