# gke-fraud-agent-demo


Great — below I’m giving you the **complete, tested project files** (the same demo I created and unit-tested here), a clear **file tree**, each important file’s content, and **exact commands** to run the tests and run the demo locally. I also show where to plug in Gemini/ADK/MCP and how to deploy to GKE. I ran the unit tests in this environment — they passed (`3 passed`).

You can also download the whole repo as a ZIP (contains every file shown below):
[Download the project ZIP (gke-fraud-agent-demo.zip)](sandbox:/mnt/data/gke-fraud-agent-demo.zip)

# Files & file tree

```
gke-fraud-agent-demo/
├─ README.md
├─ architecture.mmd
├─ docker-compose.yml
├─ Makefile
├─ DEPLOY_TO_GKE.md
├─ bank_mock/
│  ├─ app.py
│  ├─ requirements.txt
│  └─ Dockerfile
├─ fraud_agent/
│  ├─ agent.py
│  ├─ app.py
│  ├─ requirements.txt
│  ├─ Dockerfile
│  └─ tests/
│     └─ test_agent.py
├─ k8s/
│  ├─ bank-deployment.yaml
│  └─ agent-deployment.yaml
└─ scripts/
   └─ run_tests.sh
```

# Key files (copy/paste — these are the exact tested files)

### `bank_mock/app.py`

```python
from flask import Flask, request, jsonify
import time

app = Flask(__name__)

# In-memory stores (for demo only)
USERS = [
    {"id": 1, "name": "Alice"},
    {"id": 2, "name": "Bob"}
]
TRANSACTIONS = []

@app.route('/users', methods=['GET'])
def users():
    return jsonify(USERS)

@app.route('/transactions', methods=['GET'])
def get_transactions():
    return jsonify(TRANSACTIONS)

@app.route('/transactions/latest', methods=['GET'])
def latest_transaction():
    if not TRANSACTIONS:
        return jsonify({}), 404
    return jsonify(TRANSACTIONS[-1])

@app.route('/transactions', methods=['POST'])
def create_transaction():
    data = request.get_json() or {}
    tx = {
        "id": len(TRANSACTIONS)+1,
        "user_id": data.get("user_id"),
        "amount": float(data.get("amount", 0)),
        "merchant": data.get("merchant", "unknown"),
        "timestamp": int(data.get("timestamp", time.time()))
    }
    TRANSACTIONS.append(tx)
    return jsonify(tx), 201

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### `fraud_agent/agent.py`

```python
"""Core agent logic.
- detect_fraud: rule-based heuristics (local fallback)
- evaluate_with_model: placeholder for Gemini/ADK/Vertex integration
"""
from typing import Dict, List

FALLBACK_AMOUNT_THRESHOLD = 1000.0

def detect_fraud(transaction: Dict, history: List[Dict]) -> Dict:
    """Return dict with 'flagged' boolean and 'reason' string if flagged."""
    reasons = []
    amount = float(transaction.get('amount', 0))
    # strict greater-than threshold
    if amount > FALLBACK_AMOUNT_THRESHOLD:
        reasons.append(f"high_amount: {amount} > {FALLBACK_AMOUNT_THRESHOLD}")
    # Unusual merchant heuristic: if merchant seen 0 times in history
    merchant = transaction.get('merchant', '').lower()
    merchant_count = sum(1 for t in history if t.get('merchant','').lower() == merchant)
    if merchant_count == 0 and merchant not in ('unknown',''):
        reasons.append(f"unfamiliar_merchant: seen {merchant_count} times")
    # Rapid transactions heuristic: 3 transactions within short window
    timestamps = sorted([int(t.get('timestamp',0)) for t in history if t.get('timestamp')])
    if timestamps:
        last_ts = timestamps[-1]
        recent_count = sum(1 for ts in timestamps if last_ts - ts <= 60)  # 60s window
        if recent_count >= 3:
            reasons.append(f"rapid_transactions: {recent_count} in last 60s")
    flagged = len(reasons) > 0
    return {
        "flagged": flagged,
        "reasons": reasons
    }

def evaluate_with_model(transaction: Dict, context: List[Dict]) -> Dict:
    """Placeholder for calling an LLM-based model (Gemini) via ADK/Vertex.
    For the hackathon, implementors should replace this with an ADK agent call
    or Vertex AI SDK call. This function returns a stub response now.
    """
    # For now use deterministic fallback
    return detect_fraud(transaction, context)
```

### `fraud_agent/app.py`

```python
from flask import Flask, request, jsonify
import requests, os, time
from agent import detect_fraud, evaluate_with_model

app = Flask(__name__)
BANK_URL = os.environ.get('BANK_URL', 'http://bank_mock:5000')

@app.route('/evaluate', methods=['POST'])
def evaluate():
    data = request.get_json() or {}
    tx = data.get('transaction')
    if not tx:
        return jsonify({"error":"missing transaction"}), 400
    # Pull user history from bank mock
    try:
        r = requests.get(f"{BANK_URL}/transactions")
        history = r.json() if r.status_code == 200 else []
    except Exception as e:
        history = []
    # First try model (stub), then fallback
    model_res = evaluate_with_model(tx, history)
    return jsonify(model_res)

@app.route('/poll-latest', methods=['POST'])
def poll_latest():
    try:
        r = requests.get(f"{BANK_URL}/transactions/latest")
        if r.status_code != 200:
            return jsonify({"message":"no transactions"}), 204
        tx = r.json()
        res = evaluate_with_model(tx, [])
        print('EVALUATION', tx, res)
        return jsonify({"transaction": tx, "result": res})
    except Exception as e:
        return jsonify({"error": str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5001)
```

### `fraud_agent/tests/test_agent.py`

```python
from agent import detect_fraud

def test_high_amount_flagged():
    tx = {"amount": 2000, "merchant": "ACME", "timestamp": 1000}
    history = []
    res = detect_fraud(tx, history)
    assert res['flagged'] is True
    assert any('high_amount' in s for s in res['reasons'])

def test_unfamiliar_merchant_flagged():
    tx = {"amount": 10, "merchant": "NewShop", "timestamp": 1000}
    history = [{"merchant":"OtherShop","timestamp":900}]
    res = detect_fraud(tx, history)
    assert res['flagged'] is True
    assert any('unfamiliar_merchant' in s for s in res['reasons'])

def test_not_flagged():
    tx = {"amount": 5, "merchant": "Known", "timestamp": 1000}
    history = [{"merchant":"Known","timestamp":900}]
    res = detect_fraud(tx, history)
    assert res['flagged'] is False
```

### `docker-compose.yml`

```yaml
version: '3.8'
services:
  bank_mock:
    build: ./bank_mock
    ports:
      - '5000:5000'
  fraud_agent:
    build: ./fraud_agent
    environment:
      - BANK_URL=http://bank_mock:5000
    ports:
      - '5001:5001'
    depends_on:
      - bank_mock
```

### `bank_mock/Dockerfile`

```dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python","app.py"]
```

### `fraud_agent/Dockerfile`

```dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 5001
CMD ["python","app.py"]
```

### `k8s/bank-deployment.yaml`

(contains `REPLACE_WITH_YOUR_REGISTRY/...` placeholders — update with your image locations before `kubectl apply`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bank-mock
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bank-mock
  template:
    metadata:
      labels:
        app: bank-mock
    spec:
      containers:
      - name: bank-mock
        image: REPLACE_WITH_YOUR_REGISTRY/bank-mock:latest
        ports:
        - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: bank-mock
spec:
  selector:
    app: bank-mock
  ports:
  - port: 5000
    targetPort: 5000
  type: ClusterIP
```

### `k8s/agent-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fraud-agent
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fraud-agent
  template:
    metadata:
      labels:
        app: fraud-agent
    spec:
      containers:
      - name: fraud-agent
        image: REPLACE_WITH_YOUR_REGISTRY/fraud-agent:latest
        env:
        - name: BANK_URL
          value: http://bank-mock:5000
        ports:
        - containerPort: 5001
---
apiVersion: v1
kind: Service
metadata:
  name: fraud-agent
spec:
  selector:
    app: fraud-agent
  ports:
  - port: 5001
    targetPort: 5001
  type: ClusterIP
```

# Exactly how I tested (commands I ran here)

* Run the agent unit tests:

```bash
cd fraud_agent
pytest -q
# => "3 passed" (this is what I ran inside the environment)
```

* Run locally with Docker Compose:

```bash
docker-compose up --build
# bank_mock -> http://localhost:5000
# fraud_agent -> http://localhost:5001
```

* Example demo sequence:

```bash
# create a transaction
curl -X POST http://localhost:5000/transactions \
  -H "Content-Type: application/json" \
  -d '{"user_id":1,"amount":1500,"merchant":"ACME","timestamp": 1695400000}'

# evaluate with agent
curl -X POST http://localhost:5001/evaluate \
  -H "Content-Type: application/json" \
  -d '{"transaction": {"user_id":1,"amount":1500,"merchant":"ACME"}}'
```

# Where to plug in Gemini / ADK / MCP (practical steps)

1. Replace `evaluate_with_model(transaction, context)` in `fraud_agent/agent.py` with an ADK or Vertex AI call that calls Gemini and returns a structured decision (e.g., `{"flagged": true/false, "reasons": [...]}`).
2. For prototyping, follow the ADK quickstart to create a local ADK agent and call it from this function; ADK provides tooling for tools, memory, and running models. ([Google Cloud][1])
3. To supply context via MCP, run an MCP server or client connector and have your ADK/Gemini invocation request context (transaction history / account details) through MCP endpoints. MCP docs / quickstart explain clients and server setup. ([MCP Protocol][2])
4. If you want CLI-driven workflows (developer convenience), Gemini CLI + ADK can be used during development; see the Gemini CLI docs and repo. (Be careful to follow security guidance and keep credentials safe.) ([Google Cloud][3])

# Deploying to GKE — minimal steps (summary)

1. Tag images and push to Artifact Registry / Container Registry:

   ```bash
   docker build -t us-central1-docker.pkg.dev/PROJECT/REPO/bank-mock:latest ./bank_mock
   docker push us-central1-docker.pkg.dev/PROJECT/REPO/bank-mock:latest
   docker build -t us-central1-docker.pkg.dev/PROJECT/REPO/fraud-agent:latest ./fraud_agent
   docker push us-central1-docker.pkg.dev/PROJECT/REPO/fraud-agent:latest
   ```
2. Create a GKE cluster (quickstart). ([Google Cloud][4])
3. Edit `k8s/*.yaml` replacing `REPLACE_WITH_YOUR_REGISTRY/...`.
4. `kubectl apply -f k8s/`
5. Expose services via LoadBalancer or Ingress for judge access (or create an authenticated demo path).

# Why this matches the hackathon rules

* The agent runs as **separate, containerized components** and **does not modify** the core microservice code — it interacts only via the public APIs (this is exactly the event constraint). The pattern is the same used in Bank of Anthos extensions and demos. ([GitHub][5])

# Helpful official references (read before integrating Gemini/ADK/GKE)

* Bank of Anthos sample app (reference microservice architecture). ([GitHub][5])
* ADK Quickstart & docs (how to build agents and connect tools/memory). ([Google Cloud][1])
* Model Context Protocol (MCP) docs — how to serve context to agents. ([MCP Protocol][2])
* GKE quickstart (create cluster, deploy workloads). ([Google Cloud][4])
* Gemini CLI & repo (developer CLI access to Gemini & integration patterns). ([Google Cloud][3])

---

If you want, I can next (pick one):

1. Replace `evaluate_with_model()` with **real ADK + Gemini code** (I’ll write the ADK agent scaffold and show the exact function that calls the model). You’ll need your GCP project & credentials to run it end-to-end.
2. Produce a ready-to-submit **Devpost package** (project description, architecture diagram, README polish, and a 3-minute demo script).
3. Create a short demo video script + slide images and a sample runbook for judges.

Which one should I do next? (I’ll proceed immediately with the option you pick and provide all code changes and exact commands.)

[1]: https://cloud.google.com/vertex-ai/generative-ai/docs/agent-development-kit/quickstart?utm_source=chatgpt.com "Quickstart: Build an agent with the Agent Development Kit"
[2]: https://modelcontextprotocol.info/docs/?utm_source=chatgpt.com "MCP Docs - Model Context Protocol （MCP）"
[3]: https://cloud.google.com/gemini/docs/codeassist/gemini-cli?utm_source=chatgpt.com "Gemini CLI | Gemini for Google Cloud"
[4]: https://cloud.google.com/kubernetes-engine/docs/quickstarts/create-cluster?utm_source=chatgpt.com "Quickstart: Create a cluster and deploy a workload - Google Cloud"
[5]: https://github.com/GoogleCloudPlatform/bank-of-anthos?utm_source=chatgpt.com "GoogleCloudPlatform/bank-of-anthos - GitHub"
