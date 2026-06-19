# Intent Classifier — EKS Deployment Notes

**Date:** 2026-06-18  
**Cluster:** demo-cluster (us-east-1)  
**Account ID:** 737971166371

---

## Architecture

```
Internet --> ALB Ingress --> ClusterIP Service --> Pod (gunicorn/Flask, port 6000)
```

| Resource | Value |
|---|---|
| ALB DNS | `k8s-intentna-intentcl-30447cc53b-2049773019.us-east-1.elb.amazonaws.com` |
| ECR Image | `737971166371.dkr.ecr.us-east-1.amazonaws.com/mlops:latest` |
| Namespace | `intent-namespace` |

### App Endpoints
| Method | Path | Description |
|---|---|---|
| GET | `/health` | Returns `{"status":"ok"}` |
| POST | `/predict` | Returns `{"intent":"<class>"}` for a given `{"text":"..."}` |

---

## Phase 1 — Initial Deployment (LoadBalancer Service)

```powershell
# Apply manifests
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# Get external IP
kubectl get svc intent-classifier -n intent-namespace
```

**Test predict endpoint:**
```powershell
curl -Method POST `
  -Uri "http://<EXTERNAL-IP>/predict" `
  -Headers @{"Content-Type"="application/json"} `
  -Body '{"text": "I want to book a flight"}'
```

> **Note:** Service type was later changed to `ClusterIP` when Ingress was introduced.

---

## Phase 2 — AWS Load Balancer Controller Setup

**Why:** One ALB handles all services (cost efficient), supports path-based routing and HTTPS/TLS via ACM — the production-grade ingress pattern for EKS.

### Step 1 — Associate OIDC Provider

> **Why OIDC?** OIDC is the trust bridge between Kubernetes and AWS IAM. Without it, pods cannot assume IAM roles to make AWS API calls (e.g. creating ALBs).

```powershell
eksctl utils associate-iam-oidc-provider --region us-east-1 --cluster demo-cluster --approve
```

### Step 2 — Download IAM Policy

```powershell
Invoke-WebRequest `
  -Uri "https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json" `
  -OutFile "iam_policy.json"
```

### Step 3 — Create IAM Policy

> **Why?** Grants the controller permissions to create/manage ALBs, target groups, listeners, and security groups.

```powershell
aws iam create-policy `
  --policy-name AWSLoadBalancerControllerIAMPolicy `
  --policy-document file://iam_policy.json
```

**Policy ARN:** `arn:aws:iam::737971166371:policy/AWSLoadBalancerControllerIAMPolicy`

### Step 4 — Create IAM Service Account

> **Why?** A Kubernetes service account annotated with the IAM role ARN. Pods automatically receive AWS credentials via OIDC token exchange — no hardcoded keys needed.

```powershell
eksctl create iamserviceaccount `
  --cluster=demo-cluster `
  --namespace=kube-system `
  --name=aws-load-balancer-controller `
  --role-name AmazonEKSLoadBalancerControllerRole `
  --attach-policy-arn=arn:aws:iam::737971166371:policy/AWSLoadBalancerControllerIAMPolicy `
  --approve
```

### Step 5 — Install Controller via Helm

> **Why?** Watches for Kubernetes Ingress resources and automatically provisions and configures an AWS ALB.

```powershell
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller `
  -n kube-system `
  --set clusterName=demo-cluster `
  --set serviceAccount.create=false `
  --set serviceAccount.name=aws-load-balancer-controller
```

**Verify controller is running:**
```powershell
kubectl get deployment -n kube-system aws-load-balancer-controller
# Expected: READY 2/2
```

### Troubleshooting — IAM Policy Missing Permission

If you see `AccessDenied for DescribeListenerAttributes` in controller logs:

```powershell
# Check logs
kubectl logs -n kube-system deployment/aws-load-balancer-controller --tail=50

# Fix: update policy to latest and restart controller
Invoke-WebRequest `
  -Uri "https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json" `
  -OutFile "iam_policy.json"

aws iam create-policy-version `
  --policy-arn arn:aws:iam::737971166371:policy/AWSLoadBalancerControllerIAMPolicy `
  --policy-document file://iam_policy.json `
  --set-as-default

kubectl rollout restart deployment aws-load-balancer-controller -n kube-system
```

---

## Phase 3 — Ingress Setup

### Step 1 — Update Service to ClusterIP (`service.yaml`)

> **Why?** With an Ingress controller handling external traffic, the service no longer needs a public IP. `ClusterIP` keeps it internal only.

```yaml
spec:
  type: ClusterIP
```

```powershell
kubectl apply -f service.yaml
```

### Step 2 — Create Ingress (`ingress.yaml`)

> **Why?** Tells the ALB controller to provision an internet-facing ALB and route all traffic to the intent-classifier service.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: intent-classifier-ingress
  namespace: intent-namespace
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: intent-classifier
                port:
                  number: 80
```

```powershell
kubectl apply -f ingress.yaml

# Wait 1-2 mins for ADDRESS to appear
kubectl get ingress -n intent-namespace
```

---

## Verification Commands

```powershell
# Check all resources in namespace
kubectl get all -n intent-namespace

# Check ingress address
kubectl get ingress -n intent-namespace

# Check controller logs
kubectl logs -n kube-system deployment/aws-load-balancer-controller --tail=50

# Health check
curl http://k8s-intentna-intentcl-30447cc53b-2049773019.us-east-1.elb.amazonaws.com/health

# Predict
curl -Method POST `
  -Uri "http://k8s-intentna-intentcl-30447cc53b-2049773019.us-east-1.elb.amazonaws.com/predict" `
  -Headers @{"Content-Type"="application/json"} `
  -Body '{"text": "I want to book a flight"}'
# Expected: {"intent":"greeting"}
```

---

## Phase 4 — Cluster Teardown (Avoid AWS Charges)

> **Important:** Always delete the Ingress first so the ALB controller can cleanly remove the ALB before the cluster is gone. Skipping this may leave an orphaned ALB still incurring charges.

### Step 1 — Delete the Ingress (removes the ALB)
```powershell
kubectl delete ingress intent-classifier-ingress -n intent-namespace
```

### Step 2 — Delete the EKS Cluster
```powershell
eksctl delete cluster --name demo-cluster --region us-east-1
```
This takes 10-15 minutes. Deletes nodes, node groups, IAM service account, OIDC provider, and control plane.

### Step 3 — Verify Everything is Deleted
```powershell
# Confirm cluster is gone
aws eks list-clusters --region us-east-1
# Expected: { "clusters": [] }

# Confirm no orphaned Load Balancers
aws elbv2 describe-load-balancers --region us-east-1 --query "LoadBalancers[*].LoadBalancerName"
# Expected: []
```

### Optional — Delete ECR Image (stops image storage charges)
```powershell
aws ecr delete-repository --repository-name mlops --region us-east-1 --force
```

---

## Git — Push to GitHub

> **Note:** When pushing from VS Code terminal, Git Credential Manager may open a browser OAuth flow that fails (`127.0.0.1` refused to connect). Fix this permanently by switching to credential store and embedding the PAT in the remote URL.

### One-time fix (run once in VS Code terminal)
```powershell
git config --global credential.helper store
git remote set-url origin https://<YOUR-PAT>@github.com/sohailjalal/mlops-zero-to-hero.git
```

### Stage and push
```powershell
git add 04-k8s-manifests/deployment.yaml
git add 04-k8s-manifests/service.yaml
git add 04-k8s-manifests/ingress.yaml
git add 04-k8s-manifests/deployment-notes.md
git add .gitignore
git commit -m "your commit message here"
git push origin kubernetes
```

> **Security:** Generate a PAT at https://github.com/settings/tokens (Settings → Developer Settings → Personal Access Tokens). Never commit the token value into code.

---

## Possible Next Steps

| Step | Description |
|---|---|
| HTTPS/TLS | Add ACM certificate and update ingress with SSL annotations |
| Custom Domain | Route 53 CNAME pointing to the ALB DNS name |
| Auto-scaling | HPA based on CPU/request load |
