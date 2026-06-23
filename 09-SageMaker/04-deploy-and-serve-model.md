# Deploy and Serve Intent Classifier Model using SageMaker

> **Prerequisites:** Model trained and `intent_model.pkl` uploaded to S3 (see `03-create-and-save-models.md`)

---

## Step 1 — Create Inference Script

Inside JupyterLab: File → New → Text File → name it `inference.py`

```python
import joblib
import os
import json

def model_fn(model_dir):
    model = joblib.load(os.path.join(model_dir, "intent_model.pkl"))
    return model

def input_fn(request_body, request_content_type):
    data = json.loads(request_body)
    return data["instances"]

def predict_fn(input_data, model):
    return model.predict(input_data)

def output_fn(prediction, content_type):
    return json.dumps({"predictions": prediction.tolist()})
```

---

## Step 2 — Package Model as tar.gz

> **Note:** SageMaker requires a `model.tar.gz` — it does NOT deploy loose files.

Run in notebook:

```python
import tarfile

with tarfile.open("model.tar.gz", "w:gz") as tar:
    tar.add("model/artifacts/intent_model.pkl", arcname="intent_model.pkl")
    tar.add("inference.py")

print("model.tar.gz created")
```

---

## Step 3 — Upload model.tar.gz to S3

```python
import boto3

s3 = boto3.client("s3", region_name="us-east-1")
bucket = "intent-classifier-sagemaker-737971166371"

s3.upload_file(
    "model.tar.gz",
    bucket,
    "model-artifacts/model.tar.gz"
)
print(f"Uploaded to s3://{bucket}/model-artifacts/model.tar.gz")
```

---

## Step 4 — Create SageMaker Model (Console)

1. AWS Console → SageMaker → Models → **Create model**
2. Model name: `intent-classifier-model`
3. Container settings:
   - Framework: `Scikit-learn`
   - Version: `1.2`
   - Model data location: `s3://intent-classifier-sagemaker-737971166371/model-artifacts/model.tar.gz`
4. IAM role: `SageMakerIntentClassifierRole`
5. Click **Create model**

---

## Step 5 — Create Endpoint Configuration (Console)

1. SageMaker → Inference → Endpoint configurations → **Create endpoint configuration**
2. Name: `intent-classifier-endpoint-config`
3. Production variant:
   - Model: `intent-classifier-model`
   - Instance type: `ml.t2.medium`
   - Initial instance count: `1`
4. Click **Create**

---

## Step 6 — Create Endpoint (Console)

1. SageMaker → Inference → Endpoints → **Create endpoint**
2. Endpoint name: `intent-classifier-endpoint`
3. Select endpoint configuration: `intent-classifier-endpoint-config`
4. Click **Create endpoint**

Wait 5-7 minutes for status to become `InService`.

---

## Step 7 — Test Endpoint

```python
import boto3, json

runtime = boto3.client("sagemaker-runtime", region_name="us-east-1")

response = runtime.invoke_endpoint(
    EndpointName="intent-classifier-endpoint",
    ContentType="application/json",
    Body=json.dumps({"instances": ["I want to book a flight"]})
)

result = json.loads(response["Body"].read().decode())
print(result)
# Expected: {"predictions": ["greeting"]}
```

---

## Teardown — Delete Endpoint (avoid charges ~$0.056/hr)

```powershell
aws sagemaker delete-endpoint --endpoint-name intent-classifier-endpoint --region us-east-1
```
