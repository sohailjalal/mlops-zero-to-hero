# Create a Kubernetes (EKS) Cluster on AWS Using eksctl  

---

## 1. Prerequisites  
You should have these installed and configured:  
- AWS CLI (`aws configure` should be completed)  
- `eksctl`  
- `kubectl`
- `helm` (needed for ALB controller install in a later step)

Install missing tools via winget (run in PowerShell):

```powershell
winget install eksctl
winget install Kubernetes.kubectl
winget install Helm.Helm
```

Verify installations:

```powershell
aws --version
eksctl version
kubectl version --client
helm version
```

> **Lesson learnt:** Verify AWS credentials before creating the cluster — saves time debugging later:
> ```powershell
> aws sts get-caller-identity
> ```

---

## 2. Create a Simple EKS Cluster

```powershell
eksctl create cluster `
  --name my-cluster `
  --region us-east-1 `
  --version 1.32 `
  --nodegroup-name standard-workers `
  --node-type t3.medium `
  --nodes 2 `
  --nodes-min 1 `
  --nodes-max 3 `
  --managed
```

This command automatically creates:  
- VPC, Subnets  
- EKS Control Plane  
- Managed Node Group  
- IAM roles & required resources  

---

## 3. Configure kubeconfig

If needed, update kubeconfig manually:

```powershell
aws eks update-kubeconfig --region us-east-1 --name my-cluster
```

Verify nodes:

```powershell
kubectl get nodes
kubectl get pods -n kube-system
```

---

## 4. Quick Verification

List clusters:

```powershell
eksctl get cluster
```

Describe cluster:

```powershell
aws eks describe-cluster --name my-cluster --region us-east-1
```

---

## 5. Delete the Cluster (Cleanup)

```powershell
eksctl delete cluster --name my-cluster --region us-east-1
```

This removes the control plane and node group to avoid unwanted AWS charges.

---

If you want, I can also share  
**• a clean eksctl YAML config file**,  
**• a version for private clusters**, or  
**• a production-ready node group setup.**
