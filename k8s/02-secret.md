# DB secret (create manually — not applied with kubectl apply)

Run after creating the namespace:

bash
kubectl create secret generic db-secret \
  --namespace=product-app \
  --from-literal=DB_USER=<your-db-user> \
  --from-literal=DB_PASSWORD=<your-db-password>

