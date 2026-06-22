# EKS Multi-Tier Kubernetes Assignment

This project deploys a Spring Boot product API on AWS EKS. The API talks to a Postgres database running inside the cluster. Traffic from the internet hits an Application Load Balancer, which forwards requests to the API pods.

## Links

**Source code**  
https://github.com/gauravdwivedi/eks-kubernetes-assignment

**Docker image**  
https://hub.docker.com/r/gaurav021997/product-api

**Live API — view product records**  
http://k8s-producta-producta-48c0ee699b-1380945497.us-east-1.elb.amazonaws.com/api/products

Open the API link above in a browser to see the product records stored in the backend database. You can also fetch a single record at `/api/products/{id}` or check service health at `/api/health`.

For step-by-step setup, local testing, EKS deployment, and tear-down instructions, see [SETUP.md](./SETUP.md).
