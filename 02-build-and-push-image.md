# Build and Push Intent Classifier Model Container to AWS ECR

This document provides the commands to build the Docker image for the Intent Classifier model and push it to AWS ECR.

> **Lesson learnt:** We use ECR (not Docker Hub) because the EKS nodes already have IAM-based ECR pull permissions — no registry credentials needed in the cluster.

---

### Step 1 — Authenticate Docker to ECR

```powershell
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 737971166371.dkr.ecr.us-east-1.amazonaws.com
```

### Step 2 — Create ECR Repository (first time only)

```powershell
aws ecr create-repository --repository-name mlops --region us-east-1
```

### Step 3 — Build the Image

Run from the project root where your `Dockerfile` is:

```powershell
docker build -t mlops:latest .
```

### Step 4 — Tag for ECR

```powershell
docker tag mlops:latest 737971166371.dkr.ecr.us-east-1.amazonaws.com/mlops:latest
```

### Step 5 — Push to ECR

```powershell
docker push 737971166371.dkr.ecr.us-east-1.amazonaws.com/mlops:latest
```

### Step 6 — Verify

```powershell
aws ecr list-images --repository-name mlops --region us-east-1
```

---

> **Note:** Ensure the EKS node group IAM role has `AmazonEC2ContainerRegistryReadOnly` policy attached, otherwise pods will fail with `ImagePullBackOff`. Get the role name with:
> ```powershell
> aws eks describe-nodegroup `
>   --cluster-name my-cluster `
>   --nodegroup-name standard-workers `
>   --region us-east-1 `
>   --query "nodegroup.nodeRole"
> ```
> Then attach:
> ```powershell
> aws iam attach-role-policy `
>   --role-name <node-group-role-name> `
>   --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
> ```
