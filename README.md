# EKS Multi-Tier Kubernetes Assignment

This project deploys a Spring Boot product API on AWS EKS. The API talks to a Postgres database running inside the cluster. Traffic from the internet hits an Application Load Balancer, which forwards requests to the API pods.

## Links

**Source code**  
https://github.com/GDCREATION/eks-kubernetes-assignment

**Docker image**  
https://hub.docker.com/r/gaurav021997/product-api

**Live API while demo run — view product records**  
http://k8s-producta-producta-48c0ee699b-1380945497.us-east-1.elb.amazonaws.com/api/products

The cluster has been stopped, re-running the cluster and the cluster would change the load balancer ingress url as current url would be now invalid. Run `kubectl get ingress -n product-app` and update the URL accordingly.
