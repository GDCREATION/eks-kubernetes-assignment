# EKS Multi-Tier Kubernetes Assignment — Setup Guide

A simple two-tier setup — Spring Boot CRUD API backed by Postgres — containerized and deployed on AWS EKS.

For project links and live API URL, see [README.md](./README.md). For requirement analysis, assumptions, and resource justification, see [DOCUMENTATION.md](./DOCUMENTATION.md).

```
         Internet
            │
     ┌──────▼───────┐
     │  ALB Ingress  │   (externally accessible)
     └──────┬───────┘
            │
     ┌──────▼────────────┐
     │  product-api pods  │  x4, HPA-managed
     │  (Spring Boot)     │
     └──────┬────────────┘
            │  ClusterIP DNS (postgres-db)
     ┌──────▼────────────┐
     │  postgres pod      │  x1, StatefulSet + EBS PVC
     └───────────────────┘
```

## Project structure

```
.
├── app/                   Spring Boot microservice
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/
├── db/
│   └── init.sql           table DDL + 6 seed rows
├── docker-compose.yml     local testing
├── eks/
│   └── cluster.yaml       eksctl cluster definition
└── k8s/
    ├── 00-namespace.yaml
    ├── 01-configmap.yaml       DB host / port / name
    ├── 02-secret.md            kubectl command to create DB secret
    ├── 03a-storageclass.yaml   gp3 StorageClass
    ├── 03-db-configmap.yaml    init.sql mounted into postgres pod
    ├── 04-db-statefulset.yaml
    ├── 05-db-service.yaml      ClusterIP (internal only)
    ├── 06-api-deployment.yaml  4 replicas, rolling updates, probes
    ├── 07-api-service.yaml     ClusterIP
    ├── 08-api-ingress.yaml     ALB Ingress
    └── 09-api-hpa.yaml         HPA (CPU-based)
```

---

## 1. Local testing with Docker Compose

Requirements: Docker Desktop (or Docker Engine + Compose plugin).

```bash
# build and start everything (uses your local CPU architecture — fine for local dev)
docker compose up --build

# test
curl http://localhost:8080/api/products
```

To stop and clean up:

```bash
docker compose down -v   # -v also removes the named volume
```

---

## 2. Build and push the Docker image

EKS nodes run on **linux/amd64**. If you build on an Apple Silicon Mac without specifying the platform, the image will be ARM64 and pods will crash with `exec format error`.

```bash
cd app

docker login   # once, before your first push

# build for EKS (required on Apple Silicon Macs)
docker buildx build --platform linux/amd64 \
  -t <your-dockerhub-username>/product-api:latest \
  --push .

# on Intel Mac / Linux you can use a plain build instead:
# docker build -t <your-dockerhub-username>/product-api:latest .
# docker push <your-dockerhub-username>/product-api:latest
```

After pushing, update the `image:` field in `k8s/06-api-deployment.yaml` with your actual username.

---

## 3. Provision the EKS cluster

Requirements: AWS CLI configured, `eksctl` installed, `kubectl` installed.

```bash
# creates the cluster, node group, and enables the EBS CSI add-on
eksctl create cluster -f eks/cluster.yaml

# verify nodes are Ready
kubectl get nodes
```

This takes around 15 minutes the first time.

### 3a. Install metrics-server (required for HPA)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# wait a minute then verify
kubectl top nodes
```

### 3b. Install the AWS Load Balancer Controller (required for ALB Ingress)

The eksctl config already grants the needed IAM permissions to the node group.

```bash
# add the Helm repo
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# install the controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=product-cluster \
  --set serviceAccount.create=true

# verify
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

## 4. Deploy to EKS

**Prerequisite:** Push the API image built for **linux/amd64** (see [section 2](#2-build-and-push-the-docker-image)). On Apple Silicon Macs, skipping `--platform linux/amd64` causes `exec format error` in pods.

Apply the manifests in order (the number prefix keeps them sorted).

**1. Create the namespace:**

```bash
kubectl apply -f k8s/00-namespace.yaml
```

**2. Create the DB secret** (see `k8s/02-secret.md` — credentials are not stored in the repo; the namespace must exist first):

```bash
kubectl create secret generic db-secret \
  --namespace=product-app \
  --from-literal=DB_USER=<your-db-user> \
  --from-literal=DB_PASSWORD=<your-db-password>
```

**3. Apply the remaining manifests:**

```bash
kubectl apply -f k8s/01-configmap.yaml
kubectl apply -f k8s/03a-storageclass.yaml
kubectl apply -f k8s/03-db-configmap.yaml
kubectl apply -f k8s/04-db-statefulset.yaml
kubectl apply -f k8s/05-db-service.yaml
kubectl apply -f k8s/06-api-deployment.yaml
kubectl apply -f k8s/07-api-service.yaml
kubectl apply -f k8s/08-api-ingress.yaml
kubectl apply -f k8s/09-api-hpa.yaml
```

Or apply all YAML manifests at once (after creating the namespace and secret above):

```bash
kubectl apply -f k8s/01-configmap.yaml \
  -f k8s/03a-storageclass.yaml \
  -f k8s/03-db-configmap.yaml \
  -f k8s/04-db-statefulset.yaml \
  -f k8s/05-db-service.yaml \
  -f k8s/06-api-deployment.yaml \
  -f k8s/07-api-service.yaml \
  -f k8s/08-api-ingress.yaml \
  -f k8s/09-api-hpa.yaml
```

### Check everything is running

```bash
# pods
kubectl get pods -n product-app

# services
kubectl get svc -n product-app

# get the ALB address (takes ~2 minutes to provision)
kubectl get ingress -n product-app
```

---

## 5. API endpoints

Replace `<ALB_ADDRESS>` with the value from `kubectl get ingress`.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/products` | list all products |
| GET | `/api/products/{id}` | get one product |
| POST | `/api/products` | create a product |
| PUT | `/api/products/{id}` | update a product |
| DELETE | `/api/products/{id}` | delete a product |
| GET | `/api/health` | health check |

Example requests:

```bash
BASE=http://<ALB_ADDRESS>

# list
curl $BASE/api/products

# create
curl -X POST $BASE/api/products \
  -H "Content-Type: application/json" \
  -d '{"name":"4K Monitor","price":349.99,"quantity":30}'

# update
curl -X PUT $BASE/api/products/1 \
  -H "Content-Type: application/json" \
  -d '{"name":"Wireless Mouse","price":24.99,"quantity":200}'

# delete
curl -X DELETE $BASE/api/products/7
```

---

## 6. Delete resources

```bash
# delete all k8s resources
kubectl delete namespace product-app

# delete the EKS cluster (Remove the EBS volumes set by storage class seperately)
eksctl delete cluster -f eks/cluster.yaml
```
