# Hands-On Lab: Containerize an ML Training + Serving Workflow

**Goal:** Build a separate training container and serving container, hand a model off between them through a registry-style artifact folder (standing in for MLflow/S3 in a classroom setting), and see why the two images are shaped so differently.
**Time:** 60–75 min
**Requirements:** Docker installed locally. **No GPU required** — this lab uses a tiny scikit-learn model so everyone can run it on a laptop CPU; GPU specifics are covered as a walkthrough (Part 4) rather than something everyone needs to run live.

---

## 0. Facilitator Setup

```
mlops-lab/
├── training/
│   ├── train.py
│   ├── requirements.txt
│   └── Dockerfile
├── serving/
│   ├── serve.py
│   ├── requirements.txt
│   └── Dockerfile
└── model-registry/        <- shared folder standing in for a real model registry
```

**`training/requirements.txt`**
```
scikit-learn==1.5.0
joblib==1.4.2
```

**`training/train.py`**
```python
from sklearn.datasets import load_iris
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
import joblib, json, os

X, y = load_iris(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

model = RandomForestClassifier(n_estimators=50, random_state=42)
model.fit(X_train, y_train)
accuracy = model.score(X_test, y_test)
print(f"Trained model accuracy: {accuracy:.3f}")

os.makedirs("/artifacts", exist_ok=True)
joblib.dump(model, "/artifacts/model.joblib")
with open("/artifacts/metrics.json", "w") as f:
    json.dump({"accuracy": accuracy}, f)
print("Model + metrics written to /artifacts")
```

**`serving/requirements.txt`**
```
scikit-learn==1.5.0
joblib==1.4.2
flask==3.0.3
```

**`serving/serve.py`**
```python
from flask import Flask, request, jsonify
import joblib

app = Flask(__name__)
model = joblib.load("/artifacts/model.joblib")

@app.route("/predict", methods=["POST"])
def predict():
    data = request.get_json()
    prediction = model.predict([data["features"]])
    return jsonify({"prediction": int(prediction[0])})

@app.route("/health")
def health():
    return {"status": "ok"}

app.run(host="0.0.0.0", port=5000)
```

---

## 1. Exercise A — Build the Training Container (15 min)

```dockerfile
# training/Dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY train.py .
CMD ["python", "train.py"]
```

```bash
cd training
docker build -t ml-train .
docker run --rm -v "$(pwd)/../model-registry:/artifacts" ml-train
```

Check that `model-registry/model.joblib` and `model-registry/metrics.json` now exist on your host. This mounted volume is standing in for "push to MLflow / S3 / a model registry" — the training container never bakes the model into its own image.

**Discussion:** this is the anti-pattern vs. best-practice slide made concrete — the model artifact lives *outside* the image, produced at run time.

---

## 2. Exercise B — Build the Serving Container (15 min)

```dockerfile
# serving/Dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY serve.py .
EXPOSE 5000
CMD ["python", "serve.py"]
```

```bash
cd ../serving
docker build -t ml-serve .
docker run --rm -p 5000:5000 -v "$(pwd)/../model-registry:/artifacts" ml-serve
```

In another terminal:
```bash
curl -X POST http://localhost:5000/predict \
  -H "Content-Type: application/json" \
  -d '{"features": [5.1, 3.5, 1.4, 0.2]}'
```

You should get back a prediction (0, 1, or 2 — the iris species class). Try `curl http://localhost:5000/health` too.

**Debrief — compare the two Dockerfiles side by side:**
- Training installed `train.py` and ran once, then exited
- Serving installs a web server and stays running, waiting for requests
- Both share the same base image and most dependencies, but they are fundamentally different *lifecycles* — this is the training-vs-serving split from the slide deck, running for real.

---

## 3. Exercise C — Simulate a Retrain-and-Redeploy Cycle (15 min)

Change `n_estimators=50` to `n_estimators=200` in `train.py`. Re-run training:

```bash
cd ../training
docker run --rm -v "$(pwd)/../model-registry:/artifacts" ml-train
```

Note the new accuracy printed. Now, **without rebuilding the serving image**, just restart the serving container — it picks up the new model file from the mounted volume automatically:

```bash
cd ../serving
docker run --rm -p 5000:5000 -v "$(pwd)/../model-registry:/artifacts" ml-serve
```

Re-run the same `curl` command. The serving container never needed a rebuild because the model is external — this is exactly why baking weights into the image is the anti-pattern: it would force a full image rebuild and push for every retrain.

---

## 4. Walkthrough (Not Hands-On): GPU Training in the Real World

Since most laptops in the room won't have a usable NVIDIA GPU, walk through this rather than running it:

```dockerfile
FROM nvidia/cuda:12.4.1-cudnn-runtime-ubuntu22.04
RUN apt-get update && apt-get install -y python3-pip
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY train.py .
CMD ["python3", "train.py"]
```

```bash
docker run --gpus all -v $(pwd)/model-registry:/artifacts ml-train-gpu
```

Point out:
- `--gpus all` requires the **NVIDIA Container Toolkit** installed on the host
- The CUDA version in the base image must be compatible with the host's installed driver
- In Kubernetes, the same access is granted via `resources.limits: { nvidia.com/gpu: 1 }` plus the NVIDIA device plugin on the cluster

If anyone in the room *does* have a working local GPU + NVIDIA Container Toolkit, they can attempt this live as a bonus.
