# Ingress Controller Options for EKS

---

> **Lesson learnt:** For EKS on AWS, the **AWS Load Balancer Controller** (ALB) is the recommended ingress controller — not Traefik. It provisions a native AWS ALB, supports IAM/OIDC authentication, and integrates cleanly with EKS. Traefik is a valid option for non-AWS clusters or when you need advanced middleware (rate limiting, auth, etc.).

---

## Option 1 — AWS Load Balancer Controller (Recommended for EKS)

See `04-k8s-manifests/deployment-notes.md` Phase 2 for the full setup steps. Summary:

```powershell
# 1. Associate OIDC provider
eksctl utils associate-iam-oidc-provider --region us-east-1 --cluster my-cluster --approve

# 2. Download IAM policy
Invoke-WebRequest `
  -Uri "https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json" `
  -OutFile "iam_policy.json"

# 3. Create IAM policy
aws iam create-policy `
  --policy-name AWSLoadBalancerControllerIAMPolicy `
  --policy-document file://iam_policy.json

# 4. Create IAM service account
eksctl create iamserviceaccount `
  --cluster=my-cluster `
  --namespace=kube-system `
  --name=aws-load-balancer-controller `
  --role-name AmazonEKSLoadBalancerControllerRole `
  --attach-policy-arn=arn:aws:iam::737971166371:policy/AWSLoadBalancerControllerIAMPolicy `
  --approve

# 5. Install via Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller `
  -n kube-system `
  --set clusterName=my-cluster `
  --set serviceAccount.create=false `
  --set serviceAccount.name=aws-load-balancer-controller

# 6. Verify
kubectl get deployment -n kube-system aws-load-balancer-controller
```

> **Important:** After a fresh cluster creation the ALB controller is NOT installed automatically — you must run these steps every time on a new cluster.

---

## Option 2 — Traefik Ingress Controller (Non-AWS or advanced routing)

```powershell
# Add Helm repo
helm repo add traefik https://helm.traefik.io/traefik
helm repo update

# Install
helm install traefik traefik/traefik `
  --namespace traefik `
  --create-namespace

# Verify
kubectl get pods -n traefik

# Get Load Balancer endpoint
kubectl get svc -n traefik
```

---

# End of installation steps
