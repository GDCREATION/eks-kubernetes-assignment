# EKS Multi-Tier Kubernetes Assignment — Documentation

## Requirement Understanding

The assignment asks for a multi-tier application deployed on Amazon EKS with clear separation between the presentation/API layer and the data layer.

**Core functional requirements**

- Build a containerized application that exposes a REST API for managing product records (create, read, update, delete).
- Back the API with a relational database that stores product data persistently.
- Deploy both tiers on Kubernetes (EKS) so the solution demonstrates production-style orchestration rather than a single standalone container.
- Make the API tier reachable from outside the cluster so records can be viewed and tested without kubectl port-forwarding.
- Keep the database tier internal — accessible only to the API pods inside the cluster, not directly from the internet.

**Non-functional and operational requirements**

- Use Docker images published to a registry (Docker Hub) so EKS nodes can pull the API image at deploy time.
- Demonstrate Kubernetes capabilities: multiple replicas, rolling updates, self-healing, health probes, autoscaling, and persistent storage for the database.
- Apply FinOps practices: define CPU/memory requests and limits, use cost-efficient storage, and configure autoscaling so capacity matches demand.
- Provide documentation covering what was built, why specific resources were chosen, and how to reproduce the deployment.

**What was delivered**

| Layer | Technology | Kubernetes resource |
|-------|------------|---------------------|
| API tier | Spring Boot 17 (Java), JPA | Deployment (4 replicas), Service, Ingress, HPA |
| Database tier | PostgreSQL 16 | StatefulSet, ClusterIP Service, EBS PVC |
| External access | AWS ALB | Ingress via AWS Load Balancer Controller |
| Configuration | ConfigMaps + Secrets | DB connection settings, credentials, init SQL |

The API serves six seed products on first startup. All CRUD endpoints are available under `/api/products`.

---

## Assumptions

The following assumptions were made while designing and implementing the solution. If any of these change, parts of the configuration may need to be updated.

**Infrastructure and cloud**

- AWS account is available with permissions to create EKS clusters, EC2 instances, EBS volumes, and Application Load Balancers.
- The cluster runs in `us-east-1`, as defined in `eks/cluster.yaml`.
- EKS worker nodes use the `linux/amd64` platform. Images built on Apple Silicon must be cross-compiled with `--platform linux/amd64` before push.
- A single managed node group is sufficient for this assignment workload; high availability across multiple availability zones is handled by EKS control plane defaults, not by spreading database replicas.

**Application and data**

- Product catalog is small (demo scale). A single Postgres instance with one replica is enough; read replicas or managed RDS are out of scope.
- Database credentials are created manually via `kubectl create secret` and are not committed to the repository.
- The `init.sql` script runs only on first database initialization (empty volume). Existing data on the EBS volume is preserved across pod restarts.
- No authentication or TLS termination is required for the assignment. The ALB exposes HTTP on port 80.

**Networking**

- The AWS Load Balancer Controller is installed separately (Helm) after cluster creation. The eksctl node group IAM policy includes `albIngress: true` so the controller can provision ALBs.
- DNS for the ALB hostname is managed by AWS. No custom domain or Route 53 record is configured.
- The Ingress routes only `/api` paths to the API service. The database service remains ClusterIP-only.

**Operations**

- `metrics-server` is installed in the cluster so the Horizontal Pod Autoscaler can read CPU metrics.
- HPA minimum replica count (2) may differ from the Deployment's initial replica count (4). Once HPA is active, it governs the running replica count based on load.
- EBS volumes use `reclaimPolicy: Retain` on the StorageClass so data survives accidental PVC deletion during development; cluster tear-down requires manual volume cleanup if Retain is kept.

**Local development**

- Docker Compose is used for local validation only. It mirrors the two-tier topology but does not replicate EKS-specific features (ALB, HPA, StatefulSet PVC binding).

---

## Solution Overview

### Architecture

Traffic flows from the internet into an internet-facing Application Load Balancer created by the Kubernetes Ingress resource. The ALB forwards HTTP requests on path `/api` to the `product-api-service` ClusterIP service, which load-balances across all healthy API pods. Each API pod connects to Postgres using the internal DNS name `postgres-db` on port 5432. The database pod mounts a dedicated EBS volume through a PersistentVolumeClaim managed by the StatefulSet.

Only the API tier is exposed externally. Postgres never receives a public IP or Ingress rule.

### Application layer

The API is a Spring Boot application packaged as a multi-stage Docker image (`app/Dockerfile`). The build stage compiles the JAR with Maven; the runtime stage uses a slim JRE image. The container listens on port 8080.

Endpoints:

- `GET /api/products` — list all products
- `GET /api/products/{id}` — get one product
- `POST /api/products` — create a product
- `PUT /api/products/{id}` — update a product
- `DELETE /api/products/{id}` — delete a product
- `GET /api/health` — liveness/readiness probe target

Database connection parameters (`DB_HOST`, `DB_PORT`, `DB_NAME`) come from a ConfigMap. Username and password come from a Kubernetes Secret.

### Database layer

Postgres runs as a StatefulSet with one replica. Stable network identity is provided by the headless-style service name `postgres-db`. Data is stored on a 5 GiB gp3 EBS volume via `volumeClaimTemplates`. The init script from `db/init.sql` is mounted through a ConfigMap into `/docker-entrypoint-initdb.d`.

Readiness and liveness probes use `pg_isready` so Kubernetes only routes traffic to the API once the database accepts connections.

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

Additional cluster components installed at deploy time:

- **metrics-server** — required for `kubectl top` and HPA CPU metrics
- **AWS Load Balancer Controller** — watches Ingress resources and creates ALBs

### Demonstrated Kubernetes features

- **Rolling updates** — Deployment strategy `RollingUpdate` with `maxSurge: 1` and `maxUnavailable: 1`; new image tags roll out without downtime.
- **Self-healing** — ReplicaSet replaces deleted API pods; StatefulSet recreates `postgres-0` if the pod is removed.
- **Autoscaling** — HPA scales API pods between 2 and 8 based on 60% average CPU utilization.
- **Persistence** — EBS volume retains Postgres data when the pod is rescheduled to another node.
- **Health checks** — HTTP probes on the API; exec probes on Postgres.

### FinOps measures

- CPU and memory requests/limits on API containers prevent noisy-neighbor issues and help the scheduler pack pods efficiently.
- HPA reduces replica count during idle periods (minimum 2).
- gp3 EBS is used instead of gp2 for lower cost per GB and baseline performance.
- Documentation in SETUP.md describes further optimizations: right-sizing from observed metrics, Spot instances, and Cluster Autoscaler.

### Repository layout

```
.
├── app/                    Spring Boot API source and Dockerfile
├── db/init.sql             Schema and seed data
├── docker-compose.yml      Local two-tier testing
├── eks/cluster.yaml        EKS cluster definition
├── k8s/                    Kubernetes manifests (numbered apply order)
├── README.md               Project links and quick reference
├── DOCUMENTATION.md        This file
└── SETUP.md                Step-by-step build and deploy guide
```

---

## Justification for the Resources Utilized

Each resource choice balances assignment requirements, operational correctness, and cost awareness.

### EKS cluster and node group

**Choice:** EKS with a managed node group of `t3.medium` instances (min 2, max 4, desired 2).

**Why:** EKS satisfies the requirement to run workloads on Kubernetes in AWS without self-managing the control plane. `t3.medium` (2 vCPU, 4 GiB RAM) provides enough capacity for several API pods plus one Postgres instance on a small demo workload. Burstable T3 instances are cost-effective for intermittent assignment traffic. The node group scales up to 4 nodes when HPA adds pods that no longer fit on existing nodes.

### API Deployment — 4 replicas, rolling update strategy

**Choice:** Deployment with 4 initial replicas, `RollingUpdate` with at most one pod unavailable at a time.

**Why:** Multiple replicas demonstrate load distribution and high availability at the API tier. Four replicas show the Deployment controller managing a non-trivial replica set while still fitting on two nodes. Rolling updates allow image upgrades without taking the entire API offline — a core Kubernetes production pattern.

### API resource requests and limits

**Choice:** Requests `250m` CPU / `256Mi` memory; limits `500m` CPU / `512Mi` memory.

**Why:** Requests tell the scheduler how much capacity to reserve per pod, which prevents overcommitting nodes. Limits cap maximum usage so a misbehaving pod cannot starve others. These values are conservative starting points for a Spring Boot JVM app; SETUP.md explains how to right-size after observing actual usage with `kubectl top pods`.

### Horizontal Pod Autoscaler

**Choice:** HPA v2 targeting CPU at 60% utilization, min 2 / max 8 replicas, faster scale-down stabilization (30s) for demo visibility.

**Why:** HPA is required to show elastic capacity. CPU is a practical default metric for a synchronous REST API under HTTP load. Minimum of 2 keeps at least one pod available during scale-down and rolling updates. Maximum of 8 prevents uncontrolled scale-up on a small cluster. The shortened scale-down window makes autoscaling behavior easier to observe during load tests without waiting five minutes.

### Postgres StatefulSet (not Deployment)

**Choice:** StatefulSet with `volumeClaimTemplates` and a stable service name.

**Why:** StatefulSets are the standard pattern for single-instance databases in Kubernetes. They provide predictable pod naming (`postgres-0`), ordered lifecycle, and per-pod persistent storage. A Deployment with a shared PVC would not bind storage correctly for ReadWriteOnce volumes across replicas.

### EBS gp3 storage — 5 GiB PVC, Retain reclaim policy

**Choice:** Custom StorageClass `gp3` via the AWS EBS CSI driver; 5 GiB volume per Postgres instance; `reclaimPolicy: Retain`.

**Why:** gp3 offers lower cost than gp2 with configurable IOPS and throughput. Five gibibytes is more than enough for the demo schema and leaves room for growth. `WaitForFirstConsumer` ensures the volume is created in the same AZ as the pod. Retain protects assignment data if the PVC is deleted by mistake; the trade-off is manual cleanup on full environment tear-down.

### ClusterIP services and Ingress (not LoadBalancer on API Service directly)

**Choice:** Internal ClusterIP for both API and DB services; external access only through Ingress + ALB.

**Why:** ClusterIP keeps the database off the public network — a basic security requirement for multi-tier designs. Using Ingress instead of `type: LoadBalancer` on the API Service gives path-based routing (`/api`), integrates with AWS ALB features, and matches common EKS production patterns. The AWS Load Balancer Controller watches Ingress annotations and provisions the ALB automatically.

### ConfigMaps and Secrets

**Choice:** ConfigMap for host, port, and database name; Secret for username and password; init SQL in a separate ConfigMap.

**Why:** Separating sensitive and non-sensitive configuration follows Kubernetes best practice. Credentials never appear in Git. ConfigMaps can be updated independently of the container image. Mounting init SQL as a ConfigMap keeps schema versioning alongside other manifests without baking it into the Postgres image.

### Health probes

**Choice:** HTTP GET `/api/health` for API readiness (30s initial delay) and liveness (60s initial delay); `pg_isready` exec probes for Postgres.

**Why:** Probes ensure Kubernetes only sends traffic to ready pods and restarts unhealthy ones. The API needs a longer initial delay because Spring Boot and JVM startup take time. Postgres probes confirm the server accepts connections before the API depends on it during rollout.

### Docker image — multi-stage build, Java 17, Docker Hub

**Choice:** Multi-stage Dockerfile (Maven builder + JRE runtime); image `gaurav021997/product-api` on Docker Hub.

**Why:** Multi-stage builds keep the runtime image small (no Maven toolchain in production). Java 17 is the current LTS supported by Spring Boot 3.x. Publishing to Docker Hub lets any EKS node pull the image with standard `imagePullPolicy: Always` for repeatable deploys.

### metrics-server and AWS Load Balancer Controller

**Choice:** Install both as cluster add-ons after EKS creation.

**Why:** HPA cannot function without metrics-server providing pod CPU usage. Ingress resources with `kubernetes.io/ingress.class: alb` have no effect until the AWS Load Balancer Controller is running. These are standard dependencies for an EKS cluster that uses autoscaling and ALB ingress — not optional for this solution.

### Docker Compose for local testing

**Choice:** Two-service Compose file mirroring API + Postgres topology.

**Why:** Developers can validate application logic and database connectivity on a laptop before incurring AWS costs. Compose does not replace EKS testing but shortens the feedback loop for code changes.

---

## Quick reference links

| Item | URL |
|------|-----|
| Source code | https://github.com/gauravdwivedi/eks-kubernetes-assignment |
| Docker image | https://hub.docker.com/r/gaurav021997/product-api |
| Live API (list products) | http://k8s-producta-producta-48c0ee699b-1380945497.us-east-1.elb.amazonaws.com/api/products |

If the ALB hostname changes after redeploying the Ingress, run `kubectl get ingress -n product-app` and update the URL accordingly.
