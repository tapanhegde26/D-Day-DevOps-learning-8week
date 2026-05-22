# Week 8: AIOps on Kubernetes + RAG Implementation

## Overview

This final week covers running AI workloads on Kubernetes and building intelligent infrastructure with RAG (Retrieval-Augmented Generation) for automated troubleshooting.

---

## Day 1-2: AI in DevOps & Ollama on Kubernetes

### AIOps Use Cases

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AIOPS APPLICATIONS                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. Anomaly Detection                                                       │
│     - Detect unusual patterns in metrics                                    │
│     - Identify potential issues before they become incidents                │
│                                                                              │
│  2. Log Analysis                                                            │
│     - Automatic error classification                                        │
│     - Root cause suggestions                                                │
│     - Pattern recognition across services                                   │
│                                                                              │
│  3. Intelligent Alerting                                                    │
│     - Reduce alert fatigue                                                  │
│     - Correlate related alerts                                              │
│     - Suggest remediation steps                                             │
│                                                                              │
│  4. Capacity Planning                                                       │
│     - Predict resource needs                                                │
│     - Optimize scaling decisions                                            │
│                                                                              │
│  5. Automated Troubleshooting                                               │
│     - Chat-based incident investigation                                     │
│     - Knowledge base integration                                            │
│     - Runbook automation                                                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Deploying Ollama on Kubernetes

```yaml
# Ollama Deployment (CPU)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama
  namespace: ai
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ollama
  template:
    metadata:
      labels:
        app: ollama
    spec:
      containers:
        - name: ollama
          image: ollama/ollama:latest
          ports:
            - containerPort: 11434
          resources:
            requests:
              cpu: "2"
              memory: "8Gi"
            limits:
              cpu: "4"
              memory: "16Gi"
          volumeMounts:
            - name: ollama-data
              mountPath: /root/.ollama
          env:
            - name: OLLAMA_HOST
              value: "0.0.0.0"
      volumes:
        - name: ollama-data
          persistentVolumeClaim:
            claimName: ollama-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: ollama
  namespace: ai
spec:
  selector:
    app: ollama
  ports:
    - port: 11434
      targetPort: 11434
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ollama-pvc
  namespace: ai
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: gp3
```

### GPU-Enabled Ollama

```yaml
# Ollama with GPU
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama-gpu
  namespace: ai
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ollama-gpu
  template:
    metadata:
      labels:
        app: ollama-gpu
    spec:
      containers:
        - name: ollama
          image: ollama/ollama:latest
          ports:
            - containerPort: 11434
          resources:
            limits:
              nvidia.com/gpu: 1
          volumeMounts:
            - name: ollama-data
              mountPath: /root/.ollama
      nodeSelector:
        nvidia.com/gpu.present: "true"
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      volumes:
        - name: ollama-data
          persistentVolumeClaim:
            claimName: ollama-gpu-pvc
```

### Loading Models

```yaml
# Job to pull model
apiVersion: batch/v1
kind: Job
metadata:
  name: pull-gemma
  namespace: ai
spec:
  template:
    spec:
      containers:
        - name: pull-model
          image: curlimages/curl
          command:
            - /bin/sh
            - -c
            - |
              curl -X POST http://ollama:11434/api/pull \
                -d '{"name": "gemma:2b"}'
      restartPolicy: Never
```

---

## Day 3-4: RAG Pipeline with Qdrant

### RAG Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         RAG PIPELINE                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. Document Ingestion                                                      │
│     ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐           │
│     │  Docs   │ ──► │  Chunk  │ ──► │ Embed   │ ──► │ Store   │           │
│     │ (runbooks,│    │  Text   │     │ (nomic) │     │ (Qdrant)│           │
│     │  logs)  │     │         │     │         │     │         │           │
│     └─────────┘     └─────────┘     └─────────┘     └─────────┘           │
│                                                                              │
│  2. Query Processing                                                        │
│     ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐           │
│     │  Query  │ ──► │ Embed   │ ──► │ Search  │ ──► │ Context │           │
│     │         │     │ Query   │     │ Qdrant  │     │         │           │
│     └─────────┘     └─────────┘     └─────────┘     └─────────┘           │
│                                           │                                 │
│                                           ▼                                 │
│  3. LLM Generation                                                          │
│     ┌─────────────────────────────────────────────────────────────────┐    │
│     │  Context + Query ──► Ollama (Gemma 2B) ──► Response             │    │
│     └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Qdrant Vector Database

```yaml
# Qdrant StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: qdrant
  namespace: ai
spec:
  serviceName: qdrant
  replicas: 1
  selector:
    matchLabels:
      app: qdrant
  template:
    metadata:
      labels:
        app: qdrant
    spec:
      containers:
        - name: qdrant
          image: qdrant/qdrant:latest
          ports:
            - containerPort: 6333
              name: http
            - containerPort: 6334
              name: grpc
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
          volumeMounts:
            - name: qdrant-data
              mountPath: /qdrant/storage
  volumeClaimTemplates:
    - metadata:
        name: qdrant-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 20Gi
---
apiVersion: v1
kind: Service
metadata:
  name: qdrant
  namespace: ai
spec:
  selector:
    app: qdrant
  ports:
    - name: http
      port: 6333
    - name: grpc
      port: 6334
```

### Document Ingestion Service

```python
# ingestion/app.py
from fastapi import FastAPI, UploadFile
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct
import requests
import hashlib
from langchain.text_splitter import RecursiveCharacterTextSplitter

app = FastAPI()
qdrant = QdrantClient(host="qdrant", port=6333)
OLLAMA_URL = "http://ollama:11434"
COLLECTION_NAME = "knowledge_base"

# Create collection if not exists
try:
    qdrant.create_collection(
        collection_name=COLLECTION_NAME,
        vectors_config=VectorParams(size=768, distance=Distance.COSINE)
    )
except:
    pass

def get_embedding(text: str) -> list:
    """Get embedding from Ollama using nomic-embed-text"""
    response = requests.post(
        f"{OLLAMA_URL}/api/embeddings",
        json={"model": "nomic-embed-text", "prompt": text}
    )
    return response.json()["embedding"]

def chunk_text(text: str, chunk_size: int = 500) -> list:
    """Split text into chunks"""
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=chunk_size,
        chunk_overlap=50
    )
    return splitter.split_text(text)

@app.post("/ingest")
async def ingest_document(file: UploadFile):
    """Ingest a document into the knowledge base"""
    content = await file.read()
    text = content.decode("utf-8")
    
    chunks = chunk_text(text)
    points = []
    
    for i, chunk in enumerate(chunks):
        embedding = get_embedding(chunk)
        point_id = hashlib.md5(f"{file.filename}_{i}".encode()).hexdigest()
        
        points.append(PointStruct(
            id=point_id,
            vector=embedding,
            payload={
                "text": chunk,
                "source": file.filename,
                "chunk_index": i
            }
        ))
    
    qdrant.upsert(collection_name=COLLECTION_NAME, points=points)
    
    return {"status": "success", "chunks_ingested": len(chunks)}

@app.post("/search")
async def search(query: str, limit: int = 5):
    """Search the knowledge base"""
    query_embedding = get_embedding(query)
    
    results = qdrant.search(
        collection_name=COLLECTION_NAME,
        query_vector=query_embedding,
        limit=limit
    )
    
    return [
        {
            "text": hit.payload["text"],
            "source": hit.payload["source"],
            "score": hit.score
        }
        for hit in results
    ]
```

---

## Day 5-6: AI-Powered Health Checker

### Health Checker CronJob

```yaml
# Health checker deployment
apiVersion: batch/v1
kind: CronJob
metadata:
  name: ai-health-checker
  namespace: ai
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: health-checker
              image: ghcr.io/company/ai-health-checker:latest
              env:
                - name: OLLAMA_URL
                  value: "http://ollama:11434"
                - name: QDRANT_URL
                  value: "http://qdrant:6333"
                - name: PROMETHEUS_URL
                  value: "http://prometheus:9090"
                - name: LOKI_URL
                  value: "http://loki:3100"
                - name: SLACK_WEBHOOK
                  valueFrom:
                    secretKeyRef:
                      name: slack-webhook
                      key: url
          restartPolicy: OnFailure
```

### Health Checker Logic

```python
# health_checker/main.py
import requests
import json
from datetime import datetime, timedelta

OLLAMA_URL = os.environ["OLLAMA_URL"]
QDRANT_URL = os.environ["QDRANT_URL"]
PROMETHEUS_URL = os.environ["PROMETHEUS_URL"]
LOKI_URL = os.environ["LOKI_URL"]
SLACK_WEBHOOK = os.environ["SLACK_WEBHOOK"]

def get_metrics():
    """Fetch key metrics from Prometheus"""
    queries = {
        "error_rate": 'sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))',
        "latency_p99": 'histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))',
        "pod_restarts": 'sum(increase(kube_pod_container_status_restarts_total[1h]))',
        "memory_usage": 'sum(container_memory_usage_bytes) / sum(machine_memory_bytes)'
    }
    
    metrics = {}
    for name, query in queries.items():
        response = requests.get(
            f"{PROMETHEUS_URL}/api/v1/query",
            params={"query": query}
        )
        result = response.json()
        if result["data"]["result"]:
            metrics[name] = float(result["data"]["result"][0]["value"][1])
    
    return metrics

def get_recent_errors():
    """Fetch recent error logs from Loki"""
    query = '{level="error"}'
    end = datetime.now()
    start = end - timedelta(minutes=15)
    
    response = requests.get(
        f"{LOKI_URL}/loki/api/v1/query_range",
        params={
            "query": query,
            "start": int(start.timestamp() * 1e9),
            "end": int(end.timestamp() * 1e9),
            "limit": 100
        }
    )
    
    logs = []
    for stream in response.json().get("data", {}).get("result", []):
        for value in stream.get("values", []):
            logs.append(value[1])
    
    return logs

def search_knowledge_base(query: str):
    """Search runbooks and documentation"""
    response = requests.post(
        f"{QDRANT_URL}/search",
        json={"query": query, "limit": 3}
    )
    return response.json()

def analyze_with_llm(metrics: dict, errors: list, context: list):
    """Use LLM to analyze the situation"""
    prompt = f"""You are a Kubernetes SRE assistant. Analyze the following system state and provide insights.

## Current Metrics
- Error Rate: {metrics.get('error_rate', 'N/A'):.2%}
- P99 Latency: {metrics.get('latency_p99', 'N/A'):.2f}s
- Pod Restarts (1h): {metrics.get('pod_restarts', 'N/A')}
- Memory Usage: {metrics.get('memory_usage', 'N/A'):.2%}

## Recent Errors (last 15 min)
{chr(10).join(errors[:10])}

## Relevant Documentation
{chr(10).join([c['text'] for c in context])}

Based on this information:
1. Is there a problem? If yes, what is it?
2. What is the likely root cause?
3. What actions should be taken?

Be concise and actionable."""

    response = requests.post(
        f"{OLLAMA_URL}/api/generate",
        json={
            "model": "gemma:2b",
            "prompt": prompt,
            "stream": False
        }
    )
    
    return response.json()["response"]

def send_slack_alert(analysis: str, severity: str):
    """Send analysis to Slack"""
    color = {"critical": "danger", "warning": "warning", "info": "good"}[severity]
    
    requests.post(SLACK_WEBHOOK, json={
        "attachments": [{
            "color": color,
            "title": f"AI Health Check - {severity.upper()}",
            "text": analysis,
            "footer": "AI Health Checker",
            "ts": int(datetime.now().timestamp())
        }]
    })

def main():
    # Gather data
    metrics = get_metrics()
    errors = get_recent_errors()
    
    # Determine if there's an issue
    has_issue = (
        metrics.get("error_rate", 0) > 0.01 or
        metrics.get("latency_p99", 0) > 2 or
        metrics.get("pod_restarts", 0) > 5 or
        len(errors) > 10
    )
    
    if has_issue:
        # Search knowledge base for relevant context
        search_query = " ".join(errors[:3]) if errors else "high error rate kubernetes"
        context = search_knowledge_base(search_query)
        
        # Analyze with LLM
        analysis = analyze_with_llm(metrics, errors, context)
        
        # Determine severity
        severity = "critical" if metrics.get("error_rate", 0) > 0.05 else "warning"
        
        # Send alert
        send_slack_alert(analysis, severity)
        print(f"Alert sent: {severity}")
    else:
        print("System healthy, no alerts needed")

if __name__ == "__main__":
    main()
```

---

## Day 7-8: AI Chat Interface & Capstone

### Chat API Service

```python
# chat/app.py
from fastapi import FastAPI, WebSocket
from pydantic import BaseModel
import requests

app = FastAPI()

OLLAMA_URL = "http://ollama:11434"
QDRANT_URL = "http://qdrant:6333"

class ChatMessage(BaseModel):
    message: str
    context: list = []

def get_embedding(text: str) -> list:
    response = requests.post(
        f"{OLLAMA_URL}/api/embeddings",
        json={"model": "nomic-embed-text", "prompt": text}
    )
    return response.json()["embedding"]

def search_context(query: str) -> list:
    embedding = get_embedding(query)
    response = requests.post(
        f"{QDRANT_URL}/collections/knowledge_base/points/search",
        json={
            "vector": embedding,
            "limit": 5,
            "with_payload": True
        }
    )
    return [hit["payload"]["text"] for hit in response.json()["result"]]

@app.post("/chat")
async def chat(message: ChatMessage):
    # Get relevant context from knowledge base
    context = search_context(message.message)
    
    # Build prompt with context
    prompt = f"""You are a helpful Kubernetes and DevOps assistant. Use the following context to answer the question.

Context:
{chr(10).join(context)}

Question: {message.message}

Answer:"""

    # Generate response
    response = requests.post(
        f"{OLLAMA_URL}/api/generate",
        json={
            "model": "gemma:2b",
            "prompt": prompt,
            "stream": False
        }
    )
    
    return {
        "response": response.json()["response"],
        "context_used": context
    }

@app.websocket("/ws/chat")
async def websocket_chat(websocket: WebSocket):
    await websocket.accept()
    
    while True:
        data = await websocket.receive_text()
        
        # Get context
        context = search_context(data)
        
        # Stream response
        prompt = f"""Context: {chr(10).join(context)}

Question: {data}

Answer:"""

        response = requests.post(
            f"{OLLAMA_URL}/api/generate",
            json={
                "model": "gemma:2b",
                "prompt": prompt,
                "stream": True
            },
            stream=True
        )
        
        for line in response.iter_lines():
            if line:
                chunk = json.loads(line)
                await websocket.send_text(chunk.get("response", ""))
                if chunk.get("done"):
                    break
```

### React Dashboard

```jsx
// dashboard/src/App.jsx
import React, { useState, useEffect, useRef } from 'react';

function App() {
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState('');
  const [metrics, setMetrics] = useState({});
  const wsRef = useRef(null);

  useEffect(() => {
    // Fetch metrics
    const fetchMetrics = async () => {
      const response = await fetch('/api/metrics');
      setMetrics(await response.json());
    };
    fetchMetrics();
    const interval = setInterval(fetchMetrics, 30000);
    return () => clearInterval(interval);
  }, []);

  const sendMessage = async () => {
    if (!input.trim()) return;
    
    setMessages(prev => [...prev, { role: 'user', content: input }]);
    
    const response = await fetch('/api/chat', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ message: input })
    });
    
    const data = await response.json();
    setMessages(prev => [...prev, { role: 'assistant', content: data.response }]);
    setInput('');
  };

  return (
    <div className="app">
      <header>
        <h1>AIOps Dashboard</h1>
      </header>
      
      <div className="metrics-panel">
        <div className="metric">
          <span>Error Rate</span>
          <span>{(metrics.error_rate * 100).toFixed(2)}%</span>
        </div>
        <div className="metric">
          <span>P99 Latency</span>
          <span>{metrics.latency_p99?.toFixed(2)}s</span>
        </div>
        <div className="metric">
          <span>Pod Restarts</span>
          <span>{metrics.pod_restarts}</span>
        </div>
      </div>
      
      <div className="chat-container">
        <div className="messages">
          {messages.map((msg, i) => (
            <div key={i} className={`message ${msg.role}`}>
              {msg.content}
            </div>
          ))}
        </div>
        
        <div className="input-area">
          <input
            value={input}
            onChange={e => setInput(e.target.value)}
            onKeyPress={e => e.key === 'Enter' && sendMessage()}
            placeholder="Ask about your infrastructure..."
          />
          <button onClick={sendMessage}>Send</button>
        </div>
      </div>
    </div>
  );
}

export default App;
```

---

## Week 8 Capstone

Complete AIOps monitoring system:
- Spring Boot app with failure injection
- Python health checker CronJob
- Ollama + Gemma 2B for log analysis
- Qdrant RAG pipeline with uploadable knowledge base
- React dashboard with live AI chat
- Everything local, zero external API calls

---

## Key Takeaways

1. **Ollama** enables local LLM deployment on Kubernetes
2. **RAG** enhances LLM responses with domain knowledge
3. **Vector databases** (Qdrant) enable semantic search
4. **AI health checkers** provide intelligent alerting
5. **Chat interfaces** democratize infrastructure knowledge
6. **GPU scheduling** requires proper node configuration

---

## Course Complete!

Congratulations on completing the 8-week DevOps course! You now have:

- Deep Kubernetes knowledge from fundamentals to advanced
- Production-grade infrastructure skills with Terraform
- Service mesh and microservices expertise
- Full observability and SRE practices
- Security-first DevSecOps mindset
- Chaos engineering capabilities
- AI-powered infrastructure management

**Next Steps:**
1. Build your portfolio with the capstone projects
2. Contribute to open-source Kubernetes projects
3. Pursue CKA/CKAD/CKS certifications
4. Apply these skills in real-world scenarios
