# Train and Save Intent Classifier Model in SageMaker Studio

> **Resume point:** Domain `d-kghqdedstvfk` and user profile `intent-classifier-user` already created.
> Open Studio: AWS Console → SageMaker → Domains → intent-classifier-domain → intent-classifier-user → Launch → Studio → JupyterLab

---

## Inside JupyterLab — Open a Notebook

File → New → Notebook → Select `Python 3` kernel

---

## Cell 1 — Install Dependencies

```python
!pip install scikit-learn joblib boto3
```

---

## Cell 2 — Train the Intent Classifier

```python
import os, joblib
from sklearn.feature_extraction.text import CountVectorizer
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import Pipeline

X = ["hi", "hello", "how to reset password", "cancel my subscription", "great service"]
y = ["greeting", "greeting", "question", "complaint", "praise"]

pipeline = Pipeline([("vect", CountVectorizer()), ("clf", MultinomialNB())])
pipeline.fit(X, y)

os.makedirs("model/artifacts", exist_ok=True)
joblib.dump(pipeline, "model/artifacts/intent_model.pkl")
print("Model trained and saved")
```

---

## Cell 3 — Upload Model to S3

```python
import boto3

s3 = boto3.client("s3", region_name="us-east-1")
bucket = "intent-classifier-sagemaker-737971166371"

s3.upload_file(
    "model/artifacts/intent_model.pkl",
    bucket,
    "model-artifacts/intent_model.pkl"
)
print(f"Uploaded to s3://{bucket}/model-artifacts/intent_model.pkl")
```

---

## Cell 4 — Verify Upload

```python
response = s3.list_objects_v2(Bucket=bucket, Prefix="model-artifacts/")
for obj in response["Contents"]:
    print(obj["Key"], "-", obj["Size"], "bytes")
```

---

## Next Step — Package model for SageMaker deployment

See `04-deploy-and-serve-model.md` for packaging into `model.tar.gz` and deploying as a SageMaker endpoint.
