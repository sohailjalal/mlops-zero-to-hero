# SageMaker Production Setup using AWS CLI (Windows 11)

> **Lessons learnt:** All commands below use Windows 11 PowerShell syntax (backticks for line continuation).
> JSON parameters must be written to file using `[System.IO.File]::WriteAllText` with ASCII encoding — PowerShell's `Out-File` adds a BOM character that breaks AWS CLI JSON parsing.

---

## Step 1 — Create S3 Bucket for Model Artifacts

```powershell
aws s3 mb s3://intent-classifier-sagemaker-737971166371 --region us-east-1
```

---

## Step 2 — Create IAM Execution Role for SageMaker

Create trust.json using ASCII encoding (avoids BOM issue):

```powershell
$json = "{`"Version`":`"2012-10-17`",`"Statement`":[{`"Effect`":`"Allow`",`"Principal`":{`"Service`":`"sagemaker.amazonaws.com`"},`"Action`":`"sts:AssumeRole`"}]}"
[System.IO.File]::WriteAllText("$PWD\trust.json", $json, [System.Text.Encoding]::ASCII)
```

Create the role:

```powershell
aws iam create-role `
  --role-name SageMakerIntentClassifierRole `
  --assume-role-policy-document file://trust.json
```

Save the Role ARN from output:
`arn:aws:iam::737971166371:role/SageMakerIntentClassifierRole`

Attach policies:

```powershell
aws iam attach-role-policy `
  --role-name SageMakerIntentClassifierRole `
  --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess

aws iam attach-role-policy `
  --role-name SageMakerIntentClassifierRole `
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
```

Verify:

```powershell
aws iam list-attached-role-policies --role-name SageMakerIntentClassifierRole
```

---

## Step 3 — Get Default VPC and Subnets

> **Lesson learnt:** Default VPC may not exist if previously deleted. Recreate it with `create-default-vpc`.

```powershell
# Check for default VPC
aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" `
  --query "Vpcs[0].VpcId" --output text --region us-east-1
```

If output is `None`, recreate the default VPC:

```powershell
aws ec2 create-default-vpc --region us-east-1
```

Get subnets (replace VPC ID):

```powershell
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-002132d161e10feff" `
  --query "Subnets[*].SubnetId" --output text --region us-east-1
```

Pick any 2 subnet IDs for the domain creation.

---

## Step 4 — Create SageMaker Domain

> **Lesson learnt:** `--default-user-settings` JSON must also be written to a file using ASCII encoding — inline JSON and PowerShell variables both fail due to quote mangling.

```powershell
$settings = "{`"ExecutionRole`":`"arn:aws:iam::737971166371:role/SageMakerIntentClassifierRole`"}"
[System.IO.File]::WriteAllText("$PWD\user-settings.json", $settings, [System.Text.Encoding]::ASCII)

aws sagemaker create-domain `
  --domain-name intent-classifier-domain `
  --auth-mode IAM `
  --default-user-settings file://user-settings.json `
  --vpc-id vpc-002132d161e10feff `
  --subnet-ids subnet-0501942d3fa774aa8 subnet-0cf9a8c8e3ccd221b `
  --region us-east-1
```

Note the returned DomainId: `d-kghqdedstvfk`

Wait for domain to be InService:

```powershell
aws sagemaker describe-domain `
  --domain-id d-kghqdedstvfk `
  --region us-east-1 `
  --query "Status"
# Wait until output is "InService" before proceeding
```

---

## Step 5 — Create User Profile

> **Lesson learnt:** Create user profile ONLY after domain status is `InService` — it will fail with `ValidationException` otherwise.

```powershell
aws sagemaker create-user-profile `
  --domain-id d-kghqdedstvfk `
  --user-profile-name intent-classifier-user `
  --region us-east-1
```

---

## Step 6 — Open SageMaker Studio

1. AWS Console → SageMaker → Domains
2. Click `intent-classifier-domain`
3. Click `intent-classifier-user`
4. Click **Launch → Studio**
5. Click **JupyterLab** → **Create JupyterLab Space**
6. Name it `intent-classifier` → **Run Space** → **Open JupyterLab**

---

## Teardown (avoid charges)

```powershell
# Delete endpoint first if created
aws sagemaker delete-endpoint --endpoint-name intent-classifier-endpoint --region us-east-1

# Delete domain
aws sagemaker delete-domain --domain-id d-kghqdedstvfk --region us-east-1

# Delete IAM role
aws iam detach-role-policy --role-name SageMakerIntentClassifierRole --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess
aws iam detach-role-policy --role-name SageMakerIntentClassifierRole --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
aws iam delete-role --role-name SageMakerIntentClassifierRole
```
