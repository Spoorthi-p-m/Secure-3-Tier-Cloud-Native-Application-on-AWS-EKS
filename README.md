# EKS 3-Tier Project — End-to-End Deployment Steps

Architecture: Nginx frontend + Flask backend + RDS PostgreSQL, deployed on
EKS Fargate, exposed via a single ALB Ingress with path-based routing,
backend autoscaled with HPA, and a NetworkPolicy restricting DB access.

---

## 0. Prerequisites
- AWS CLI configured (`aws configure`)
- `kubectl`, `eksctl`, `docker` installed
- An existing EKS cluster with Fargate profile and AWS Load Balancer
  Controller already installed (reuse your existing setup from project 3)
- ECR repos created for `backend` and `frontend`

```bash
aws ecr create-repository --repository-name backend
aws ecr create-repository --repository-name frontend
```

---

## 1. Create the RDS PostgreSQL instance (~10 min, mostly waiting)

```bash
aws rds create-db-instance \
  --db-instance-identifier eks-3tier-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --master-username appadmin \
  --master-user-password "<CHOOSE_A_STRONG_PASSWORD>" \
  --allocated-storage 20 \
  --vpc-security-group-ids <YOUR_SG_ID> \
  --db-subnet-group-name <YOUR_DB_SUBNET_GROUP> \
  --publicly-accessible false
```

- Use a security group that allows inbound port 5432 **only from your EKS
  node/pod security group**, not from the internet.
- While it provisions, move on to building images.
- Once available, get the endpoint:

```bash
aws rds describe-db-instances \
  --db-instance-identifier eks-3tier-db \
  --query "DBInstances[0].Endpoint.Address" --output text
```

---

## 2. Build and push Docker images (~15 min)

```bash
aws ecr get-login-password --region <YOUR_REGION> | \
  docker login --username AWS --password-stdin <YOUR_ACCOUNT_ID>.dkr.ecr.<YOUR_REGION>.amazonaws.com

# Backend
cd backend
docker build -t backend .
docker tag backend:latest <YOUR_ACCOUNT_ID>.dkr.ecr.<YOUR_REGION>.amazonaws.com/backend:latest
docker push <YOUR_ACCOUNT_ID>.dkr.ecr.<YOUR_REGION>.amazonaws.com/backend:latest

# Frontend
cd ../frontend
docker build -t frontend .
docker tag frontend:latest <YOUR_ACCOUNT_ID>.dkr.ecr.<YOUR_REGION>.amazonaws.com/frontend:latest
docker push <YOUR_ACCOUNT_ID>.dkr.ecr.<YOUR_REGION>.amazonaws.com/frontend:latest
```

---

## 3. Fill in placeholders

Edit these files, replacing bracketed placeholders:
- `k8s/backend.yaml` → `<YOUR_ECR_REPO_URI>`
- `k8s/frontend.yaml` → `<YOUR_ECR_REPO_URI>`
- `k8s/db-secret.yaml` → `<YOUR_RDS_ENDPOINT>`, `<YOUR_DB_USERNAME>`, `<YOUR_DB_PASSWORD>`

---

## 4. Deploy to EKS (~10 min)

```bash
kubectl apply -f k8s/db-secret.yaml
kubectl apply -f k8s/backend.yaml
kubectl apply -f k8s/frontend.yaml
kubectl apply -f k8s/ingress.yaml
```

Check pods are running:

```bash
kubectl get pods
kubectl get svc
kubectl get ingress
```

Wait for the ALB address to appear under `ADDRESS` in `kubectl get ingress`
(can take 2-3 minutes). Then test:

```bash
curl http://<ALB_ADDRESS>/api/health
curl http://<ALB_ADDRESS>/
```

Open the ALB address in a browser to see the frontend UI, add an item,
and confirm it's stored via the API into RDS.

---

## 5. Install metrics-server and apply HPA (~15 min)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl apply -f k8s/hpa.yaml
kubectl get hpa
```

Generate load to trigger scaling:

```bash
kubectl run load-generator --image=busybox --restart=Never -- /bin/sh -c \
  "while true; do wget -q -O- http://backend-service:5000/api/items; done"
```

Watch scaling happen:

```bash
kubectl get hpa backend-hpa --watch
kubectl get pods -l app=backend --watch
```

Clean up the load generator afterward:

```bash
kubectl delete pod load-generator
```

---

## 6. Apply NetworkPolicy (~10 min)

```bash
kubectl apply -f k8s/network-policy.yaml
```

Verify enforcement — this should succeed (frontend calling backend):

```bash
kubectl exec -it deploy/frontend-deployment -- wget -qO- http://backend-service:5000/api/health
```

This should fail/timeout if you try from an unlabeled pod:

```bash
kubectl run test-pod --image=busybox --restart=Never -it -- \
  wget -qO- --timeout=5 http://backend-service:5000/api/health
kubectl delete pod test-pod
```

> Note: NetworkPolicy enforcement on EKS requires either the AWS VPC CNI
> with `ENABLE_NETWORK_POLICY=true` set, or Calico installed. If you're on
> Fargate, confirm your setup supports policy enforcement before relying
> on this for the demo — otherwise mention it as "configured, enforcement
> validated on CNI-supported nodes" in your interview explanation.

---

## 7. What to say about this project in an interview

- **Routing:** "A single ALB, provisioned via the AWS Load Balancer
  Controller, does path-based routing — `/` to the frontend, `/api` to
  the backend — so both tiers share one load balancer instead of one each."
- **Scaling:** "The backend has an HPA that scales pods 2→6 based on CPU
  utilization, verified by generating load and watching pods scale in
  real time."
- **Security:** "A NetworkPolicy restricts backend pod ingress to only
  the frontend, and the RDS security group only allows inbound traffic
  from the cluster's security group — not the public internet."
- **Data tier:** "The backend connects to RDS PostgreSQL using credentials
  stored in a Kubernetes Secret, keeping credentials out of the image and
  source code."

---

## 8. Cleanup (avoid ongoing AWS charges)

```bash
kubectl delete -f k8s/
aws rds delete-db-instance --db-instance-identifier eks-3tier-db --skip-final-snapshot
```
