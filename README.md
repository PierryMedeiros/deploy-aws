# Deploying Application on AWS

This repository contains all the necessary files and instructions to deploy an application on AWS as part of the company's course.

## Steps to Deploy

### 1. Set up the Cluster and Install Helm
1.1. Create the cluster:
```bash
eksctl create cluster -f cluster.yaml
```

1.2. Add the Bitnami Helm repository:
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

1.3. Install the RabbitMQ Operator:
```bash
helm install rabbitmq-operator bitnami/rabbitmq-cluster-operator
```

Wait for the installation to complete.

---

### 2. Deploy Namespace
```bash
kubectl apply -f namespace.yaml
```

---

### 3. Deploy StorageClass
```bash
kubectl apply -f storageclass.yaml
```

---

### 4. Deploy RabbitMQ
4.1. Apply the RabbitMQ configuration:
```bash
kubectl apply -f rabbitmq.yaml
```

Verify availability:
- Wait until the pods are available:
  ```bash
  kubectl get all -l app.kubernetes.io/name=rabbitmq -n codeflix
  ```
- Ensure the status is `true`:
  ```bash
  kubectl describe rmq rabbitmq -n codeflix
  ```

If it takes too long, check the logs for errors:
```bash
kubectl describe pod/rabbitmq-server-0 -n codeflix
```

Test RabbitMQ access with port forwarding:
```bash
kubectl port-forward "service/rabbitmq" 15672 -n codeflix
```

Retrieve the RabbitMQ username and password:
```bash
username="$(kubectl get secret rabbitmq-default-user -o jsonpath='{.data.username}' -n codeflix | base64 --decode)"
echo "username: $username"
password="$(kubectl get secret rabbitmq-default-user -o jsonpath='{.data.password}' -n codeflix | base64 --decode)"
echo "password: $password"
```

Create the RabbitMQ secret:
```bash
kubectl create secret generic rabbitmq-secret \
--from-literal=username=adm_videos \
--from-literal=password=123456 -n codeflix
```

Apply the RabbitMQ topology:
```bash
kubectl apply -f rabbitmq-topology.yaml
```

---

### 5. Deploy Database and Service
Deploy the database:
```bash
kubectl apply -f db.yaml
```

Wait for the database to start, then deploy the service:
```bash
kubectl apply -f db-service.yaml
```

---

### 6. Deploy Keycloak

Install the Keycloak Operator:
```bash
kubectl apply -f https://raw.githubusercontent.com/keycloak/keycloak-k8s-resources/26.1.4/kubernetes/keycloaks.k8s.keycloak.org-v1.yml
kubectl apply -f https://raw.githubusercontent.com/keycloak/keycloak-k8s-resources/26.1.4/kubernetes/keycloakrealmimports.k8s.keycloak.org-v1.yml
kubectl apply -f https://raw.githubusercontent.com/keycloak/keycloak-k8s-resources/26.1.4/kubernetes/kubernetes.yml -n codeflix
```

Create the database secrets:
```bash
kubectl create secret generic db-secret \
--from-literal=username=root \
--from-literal=password=root -n codeflix
```

Deploy Keycloak:
```bash
kubectl apply -f keycloak.yaml
```

Check Keycloak readiness:
```bash
kubectl get keycloaks/keycloak -n codeflix \
  -o go-template='{{range .status.conditions}}CONDITION: {{.type}}{{"\n"}}  STATUS: {{.status}}{{"\n"}}  MESSAGE: {{.message}}{{"\n"}}{{end}}'
```

Apply the Keycloak realm configuration:
```bash
kubectl apply -f keycloak-import.yaml
```

Monitor the realm import:
```bash
kubectl get keycloakrealmimports/kc-codeflix-realm -n codeflix -o go-template='{{range .status.conditions}}CONDITION: {{.type}}{{"\n"}}  STATUS: {{.status}}{{"\n"}}  MESSAGE: {{.message}}{{"\n"}}{{end}}'
```

When `done` is `true`, the import was successful.

Bind the port and retrieve the Keycloak admin credentials:
```bash
kubectl port-forward service/keycloak-service 8080:80 -n codeflix

kubectl get secret example-kc-initial-admin -o jsonpath='{.data.username}' | base64 --decode
kubectl get secret example-kc-initial-admin -o jsonpath='{.data.password}' | base64 --decode
```

---

### 7. Deploy Admin Catalog
Create the GCP credentials secret:
```bash
kubectl create secret generic gcp-credentials-secret \
  --from-file=gcp_credentials.json=gcp_credentials.json -n codeflix
```

Apply the admin catalog configuration:
```bash
kubectl apply -f admin-catalog.yaml
```

Verify the application status:
```bash
kubectl get all -n codeflix
```
