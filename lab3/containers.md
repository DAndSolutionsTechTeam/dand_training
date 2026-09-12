# Hands-On Lab: Containerize a Multi-Service Project

**Goal:** Write multi-stage Dockerfiles for two different language stacks, wire them together with docker-compose, shrink an image, and push to a registry.
**Time:** 60–75 min
**Requirements:** Docker Desktop (or Docker Engine) installed locally, a free GitHub/Docker Hub account for the registry step.

---

## 0. Facilitator Setup

Give participants (or have them create) this structure:

```
multi-service-lab/
├── services/
│   ├── api/          (Node.js)
│   │   ├── package.json
│   │   ├── server.js
│   │   └── Dockerfile   <- they write this
│   └── worker/        (Python)
│       ├── requirements.txt
│       ├── worker.py
│       └── Dockerfile   <- they write this
└── docker-compose.yml   <- they write this
```

**`services/api/package.json`**
```json
{ "name": "api", "version": "1.0.0", "main": "server.js",
  "dependencies": { "express": "^4.19.0" } }
```

**`services/api/server.js`**
```javascript
const express = require("express");
const app = express();
app.get("/", (req, res) => res.send("API is alive"));
app.listen(8000, () => console.log("API listening on 8000"));
```

**`services/worker/requirements.txt`**
```
schedule==1.2.1
```

**`services/worker/worker.py`**
```python
import schedule, time

def job():
    print("worker: doing background work...")

schedule.every(5).seconds.do(job)
print("worker started")
while True:
    schedule.run_pending()
    time.sleep(1)
```

---

## 1. Exercise A — Naive Dockerfile First (10 min)

Have everyone write the simplest possible Dockerfile for the API, no multi-stage:

```dockerfile
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "server.js"]
```

```bash
cd services/api
docker build -t api-naive .
docker images api-naive
```

Write down the image size shown by `docker images`. (Expect several hundred MB — this is the baseline everyone will improve on.)

---

## 2. Exercise B — Multi-Stage & Slim Base (15 min)

Now rewrite it properly:

```dockerfile
# ---- deps stage ----
FROM node:20-slim AS deps
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev

# ---- runtime stage ----
FROM node:20-slim
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
EXPOSE 8000
CMD ["node", "server.js"]
```

```bash
docker build -t api-slim .
docker images | grep api
```

Compare sizes side by side. Discuss: what did switching to `-slim` save vs. what did multi-stage save? (In this trivial example the gain is modest since there's no compiled build step — the real payoff shows up in Exercise C with Python, and dramatically in compiled languages like Go/Java, per the slide deck.)

---

## 3. Exercise C — A Different Stack: Python Worker (15 min)

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "worker.py"]
```

```bash
cd ../worker
docker build -t worker .
docker run --rm worker
```

You should see `worker started` and periodic `worker: doing background work...` lines. Ctrl+C to stop.

**Discussion point:** the API and worker have completely different base images, dependency managers, and runtimes — but both follow the same shape (small base → copy deps → copy code → run). That consistency of *pattern*, not of *tooling*, is what scales across a polyglot repo.

---

## 4. Exercise D — Wire It Together With Compose (15 min)

At the repo root, write `docker-compose.yml`:

```yaml
services:
  api:
    build: ./services/api
    ports:
      - "8000:8000"
  worker:
    build: ./services/worker
```

```bash
docker compose up --build
```

Open `http://localhost:8000` — you should see "API is alive". Watch both services' logs interleaved in the same terminal. `Ctrl+C`, then `docker compose down`.

**Add a third service live:** have the group suggest a `redis` service and add it in under a minute:
```yaml
  redis:
    image: redis:7
```
Point out: no Dockerfile needed for services already published as images — only your own code needs a Dockerfile.

---

## 5. Exercise E — Tag & Push to a Registry (10 min)

```bash
docker build -t ghcr.io/<your-username>/api:$(git rev-parse --short HEAD) ./services/api
echo $GITHUB_TOKEN | docker login ghcr.io -u <your-username> --password-stdin
docker push ghcr.io/<your-username>/api:$(git rev-parse --short HEAD)
```

(Docker Hub works the same way with `docker login` and a `docker.io/<username>/api` tag if GHCR auth is awkward in the room.)

**Debrief:** why tag with a Git SHA instead of just `latest`? (Answer: reproducibility — you can always trace a running container back to the exact commit that built it.)

