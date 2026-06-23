# EKS Kubernetes Assignment — Documentation

**Infrastructure and cloud**

- AWS account is available with permissions to create EKS clusters, EC2 instances, EBS volumes, and Application Load Balancers.
- The cluster runs in `us-east-1`, as defined in `eks/cluster.yaml`.

**Application and data**

- A single Postgres instance with one replica, RDS are out of scope.
- Database credentials are created manually via `kubectl create secret`
- The `init.sql` script runs only on first database initialization. Existing data on the EBS volume is preserved across pod restarts.
- The ALB exposes HTTP on port 80.

**Networking**

- The AWS Load Balancer Controller is installed separately (Helm) after cluster creation.
- DNS for the ALB hostname is managed by AWS. No Route 53 record configured.
- The Ingress routes only `/api` paths to the API service. The database service uses ClusterIP.

**Operations**

- `metrics-server` is installed in the cluster so the Horizontal Pod Autoscaler can read CPU metrics.
- EBS volumes use `reclaimPolicy: Retain` on the StorageClass so data survives accidental PVC deletion during development.
**Local development**

- Docker Compose is used for local use only.

---

## Solution Overview


### Application layer

The API is a Spring Boot application packaged as a multi-stage Docker image (`app/Dockerfile`). The build stage compiles the JAR with Maven; the runtime stage uses a slim JRE image. The container listens on port 8080.


Database connection parameters (`DB_HOST`, `DB_PORT`, `DB_NAME`) come from a ConfigMap. Username and password come from a Kubernetes Secret.

### Database layer

Postgres runs as a StatefulSet with one replica. Stable network identity is provided by the headless-style service name `postgres-db`. Data is stored on a 5 GiB gp3 EBS volume via `volumeClaimTemplates`. The init script from `db/init.sql` is mounted through a ConfigMap into `/docker-entrypoint-initdb.d`.


### Kubernetes resources (deploy order)

| File | Resource | Purpose |
|------|----------|---------|
| `00-namespace.yaml` | Namespace `product-app` | Isolates all application resources |
| `01-configmap.yaml` | ConfigMap `db-config` | Non-sensitive DB connection settings |
| `02-secret.md` | (manual step) | Documents secret creation for DB credentials |
| `03a-storageclass.yaml` | StorageClass `gp3` | EBS gp3 provisioning via CSI driver |
| `03-db-configmap.yaml` | ConfigMap `db-init-sql` | Embeds init.sql for first-time DB setup |
| `04-db-statefulset.yaml` | StatefulSet `postgres` | Single Postgres instance with PVC |
| `05-db-service.yaml` | Service `postgres-db` | Internal ClusterIP on port 5432 |
| `06-api-deployment.yaml` | Deployment `product-api` | API pods with probes and resource limits |
| `07-api-service.yaml` | Service `product-api-service` | ClusterIP, port 80 → 8080 |
| `08-api-ingress.yaml` | Ingress | Internet-facing ALB, path `/api` |
| `09-api-hpa.yaml` | HPA | CPU-based scale 2–8 replicas |

### EKS cluster

The cluster is defined in `eks/cluster.yaml` using eksctl:

- Cluster name: `product-cluster`
- Kubernetes version: 1.32
- Managed node group: `t3.medium`, 2–4 nodes, 20 GiB gp3 root volumes
- Add-on: `aws-ebs-csi-driver` for dynamic EBS volume provisioning

---

## Justification

### EKS cluster and node group

**Choice:** EKS with a managed node group of `t3.medium` instances (min 2, max 4, desired 2).

**Why:** EKS satisfies the requirement to run workloads on Kubernetes in AWS without self-managing the control plane. `t3.medium` (2 vCPU, 4 GiB RAM) provides enough capacity for several API pods plus one Postgres instance for demo work.

### API Deployment — 4 replicas, rolling update strategy

### API resource requests and limits

**Choice:** Requests `250m` CPU / `256Mi` memory; limits `500m` CPU / `512Mi` memory.

**Why:** Requests tell the scheduler how much capacity to reserve per pod, which prevents overcommitting nodes. Limits cap maximum usage so a misbehaving pod cannot affect others.

### Horizontal Pod Autoscaler

**Choice:** HPA v2 targeting CPU at 60% utilization, min 2 / max 8 replicas, faster scale-down stabilization (30s) for demo purpose.

### Postgres StatefulSet (not Deployment)

**Choice:** StatefulSet with `volumeClaimTemplates` and a stable service name.

**Why:** StatefulSets are the standard pattern for single-instance databases in Kubernetes. A Deployment with a shared PVC would not bind storage correctly for ReadWriteOnce volumes across replicas.

### EBS gp3 storage — 5 GiB PVC, Retain reclaim policy

**Choice:** Custom StorageClass `gp3` via the AWS EBS CSI driver; 5 GiB volume per Postgres instance; `reclaimPolicy: Retain`.

**Why:** gp3 offers lower cost than gp2 with configurable IOPS and throughput. Five gibibytes is more than enough for the demo schema and leaves room for growth. `WaitForFirstConsumer` ensures the volume is created in the same AZ as the pod. Retain protects assignment data if the PVC is deleted by mistake; the trade-off is manual cleanup on full environment tear-down.

### ClusterIP services and Ingress (not LoadBalancer on API Service directly)

**Choice:** Internal ClusterIP for both API and DB services; external access only through Ingress + ALB.

**Why:** ClusterIP keeps the database off the public network.

### ConfigMaps and Secrets

**Choice:** ConfigMap for host, port, and database name; Secret for username and password; init SQL in a separate ConfigMap.

**Why:** Separating sensitive and non-sensitive configuration follows Kubernetes best practice. Credentials never appear in Git. ConfigMaps can be updated independently of the container image. Mounting init SQL as a ConfigMap keeps schema versioning alongside other manifests without baking it into the Postgres image.

### metrics-server and AWS Load Balancer Controller

**Choice:** Install both as cluster add-ons after EKS creation.

**Why:** HPA cannot function without metrics-server providing pod CPU usage. Ingress resources with `kubernetes.io/ingress.class: alb` have no effect until the AWS Load Balancer Controller is running.

---

## Quick reference link

| Item | URL |
|------|-----|
| Live API (list products) | http://k8s-producta-producta-48c0ee699b-1380945497.us-east-1.elb.amazonaws.com/api/products |

If the ALB hostname changes after redeploying the Ingress, run `kubectl get ingress -n product-app` and update the URL accordingly.
