# Week 19: AI Platform Engineering

## 🎯 Learning Objectives

By the end of this week, you will:
- Deploy your AI services as production-ready FastAPI applications with Docker
- Add Prometheus metrics for latency, cost, token usage, and quality tracking
- Implement circuit breakers to handle LLM API outages gracefully
- Build a cost tracking and budget enforcement system
- Create a Grafana dashboard definition for monitoring your AI systems
- Understand AI-specific production concerns that traditional services don't have

---

## 📖 Theory (20 min)

### AI Services Are Different

You've deployed REST APIs before. AI services have unique production concerns:

| Concern | Traditional API | AI Service |
|---------|----------------|-----------|
| Latency | Predictable (10-50ms) | Variable (500ms-30s depending on model, tokens) |
| Cost | Fixed per request (~free) | Variable per request ($0.001-$0.50) |
| Failure mode | 500 error, retry | Hallucination — LOOKS correct but ISN'T |
| Output size | Deterministic | Non-deterministic (same input → different output) |
| Rate limits | Your infra limits | Upstream API rate limits (Anthropic, OpenAI) |
| Scaling | CPU/memory based | Token throughput based |
| Monitoring | Status codes, latency | + token usage, cost, output quality, hallucination rate |

### The AI Service Monitoring Stack

```
┌──────────────────────────────────────────────┐
│  YOUR AI SERVICE (FastAPI)                    │
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │ REQUEST METRICS                           │ │
│  │ • Latency histogram (by endpoint, model)  │ │
│  │ • Request count (by status, endpoint)     │ │
│  │ • Active requests gauge                   │ │
│  └──────────────────────────────────────────┘ │
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │ AI-SPECIFIC METRICS                       │ │
│  │ • Tokens per request (input + output)     │ │
│  │ • Cost per request (in USD)               │ │
│  │ • Model used (haiku vs sonnet)            │ │
│  │ • Cache hit rate                          │ │
│  │ • Tool calls per investigation            │ │
│  │ • "I don't know" rate (RAG quality)       │ │
│  └──────────────────────────────────────────┘ │
│                                                │
│  ┌──────────────────────────────────────────┐ │
│  │ SAFETY METRICS                            │ │
│  │ • Circuit breaker state (open/closed)     │ │
│  │ • Budget remaining (daily/monthly)        │ │
│  │ • Rate limit headroom                     │ │
│  │ • Error rate to upstream (Anthropic API)  │ │
│  └──────────────────────────────────────────┘ │
└──────────────────────────────────────────────┘
         ↓                    ↓
    Prometheus            Structured Logs
         ↓                    ↓
      Grafana            Log aggregation
```

### Circuit Breakers for LLM APIs

The Anthropic API can have outages or rate limits. Without a circuit breaker, your service blocks:

```
Without circuit breaker:
  Request 1 → Anthropic API (timeout 30s) → 30s wasted
  Request 2 → Anthropic API (timeout 30s) → 30s wasted
  Your service: threads blocked, users waiting

With circuit breaker:
  Request 1 → Anthropic API (timeout) → FAIL → circuit OPENS
  Request 2 → Circuit OPEN → instant fallback (cached/degraded)
  ... 30s later → circuit HALF-OPEN → test one request → success → CLOSE
```

### Cost Management

AI costs can spike unexpectedly:

```
Normal day:    50 incidents × $0.10 = $5/day
Traffic spike: 500 incidents × $0.10 = $50/day
Agent bug:     50 incidents × $2.00 (stuck loop) = $100/day 😱

Defense layers:
  1. Per-request cost limit ($0.50 max)
  2. Per-hour budget ($10/hour)
  3. Daily budget ($50/day)
  4. Monthly budget ($500/month)
```

---

## 🔨 Hands-On (50 min)

### Exercise 1: Production AI Service with Full Observability (25 min)

**Goal:** FastAPI service with Prometheus metrics, circuit breaker, cost tracking, structured logging.

Create `week19/ai_service.py`:

```python
"""
Week 19, Exercise 1: Production AI Service

FastAPI service with:
- Prometheus metrics (latency, tokens, cost, cache hits)
- Structured JSON logging
- Circuit breaker for upstream API
- Cost tracking with budget limits
- Health checks (readiness + liveness)
"""
import time
import uuid
import json
import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse
from pydantic import BaseModel
from typing import Optional

import anthropic
from prometheus_client import (
    Counter, Histogram, Gauge, Info,
    generate_latest, CONTENT_TYPE_LATEST
)
from starlette.responses import Response


# ============================================================
# STRUCTURED LOGGING
# ============================================================

class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_data = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "message": record.getMessage(),
        }
        if hasattr(record, 'extra_data'):
            log_data.update(record.extra_data)
        return json.dumps(log_data)

handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())
logger = logging.getLogger("ai-service")
logger.addHandler(handler)
logger.setLevel(logging.INFO)


# ============================================================
# PROMETHEUS METRICS
# ============================================================

REQUEST_COUNT = Counter('ai_requests_total', 'Total requests', ['endpoint', 'status'])
REQUEST_LATENCY = Histogram('ai_request_seconds', 'Latency', ['endpoint'],
    buckets=[0.1, 0.5, 1.0, 2.0, 5.0, 10.0, 30.0])
ACTIVE_REQUESTS = Gauge('ai_active_requests', 'Active requests')
TOKENS_USED = Counter('ai_tokens_total', 'Tokens consumed', ['model', 'direction'])
COST_USD = Counter('ai_cost_usd_total', 'Cost in USD', ['model'])
CACHE_OPS = Counter('ai_cache_ops_total', 'Cache operations', ['result'])
NO_ANSWER = Counter('ai_no_answer_total', 'Queries with no answer')
AGENT_ITERATIONS = Histogram('ai_agent_iterations', 'Agent iterations', buckets=[1,2,3,4,5,6,8,10])

SERVICE_INFO = Info('ai_service', 'Service info')
SERVICE_INFO.info({'version': '1.0.0', 'model_primary': 'claude-sonnet-4-5-20250514'})


# ============================================================
# COST TRACKER
# ============================================================

class CostTracker:
    PRICING = {
        "claude-sonnet-4-5-20250514": {"input": 3.0, "output": 15.0},
        "claude-haiku-4-5-20241022": {"input": 0.25, "output": 1.25},
    }
    
    def __init__(self, hourly_limit=10.0, daily_limit=50.0):
        self.hourly_limit = hourly_limit
        self.daily_limit = daily_limit
        self.hourly_spend = 0.0
        self.daily_spend = 0.0
        self.total_spend = 0.0
        self.last_hour_reset = time.time()
        self.last_day_reset = time.time()
    
    def _maybe_reset(self):
        now = time.time()
        if now - self.last_hour_reset > 3600:
            self.hourly_spend = 0.0
            self.last_hour_reset = now
        if now - self.last_day_reset > 86400:
            self.daily_spend = 0.0
            self.last_day_reset = now
    
    def check_budget(self):
        self._maybe_reset()
        if self.hourly_spend >= self.hourly_limit:
            return False, f"Hourly limit: ${self.hourly_spend:.2f}>=${self.hourly_limit:.2f}"
        if self.daily_spend >= self.daily_limit:
            return False, f"Daily limit: ${self.daily_spend:.2f}>=${self.daily_limit:.2f}"
        return True, "OK"
    
    def record(self, model, input_tokens, output_tokens):
        p = self.PRICING.get(model, {"input": 3.0, "output": 15.0})
        cost = (input_tokens * p["input"] + output_tokens * p["output"]) / 1_000_000
        self.hourly_spend += cost
        self.daily_spend += cost
        self.total_spend += cost
        TOKENS_USED.labels(model=model, direction="input").inc(input_tokens)
        TOKENS_USED.labels(model=model, direction="output").inc(output_tokens)
        COST_USD.labels(model=model).inc(cost)
        return cost
    
    def get_status(self):
        self._maybe_reset()
        return {
            "hourly_spend": round(self.hourly_spend, 4),
            "hourly_remaining": round(self.hourly_limit - self.hourly_spend, 4),
            "daily_spend": round(self.daily_spend, 4),
            "daily_remaining": round(self.daily_limit - self.daily_spend, 4),
            "total_spend": round(self.total_spend, 4),
        }


# ============================================================
# CIRCUIT BREAKER
# ============================================================

class CircuitBreaker:
    def __init__(self, failure_threshold=3, recovery_timeout=30.0):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.state = "CLOSED"
        self.failure_count = 0
        self.last_failure_time = 0
        self.half_open_successes = 0
    
    def can_proceed(self):
        if self.state == "CLOSED":
            return True
        if self.state == "OPEN":
            if time.time() - self.last_failure_time >= self.recovery_timeout:
                self.state = "HALF_OPEN"
                logger.info("Circuit breaker → HALF_OPEN")
                return True
            return False
        return True  # HALF_OPEN
    
    def record_success(self):
        if self.state == "HALF_OPEN":
            self.half_open_successes += 1
            if self.half_open_successes >= 2:
                self.state = "CLOSED"
                self.failure_count = 0
                self.half_open_successes = 0
                logger.info("Circuit breaker → CLOSED")
        else:
            self.failure_count = 0
    
    def record_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = "OPEN"
            logger.warning(f"Circuit breaker → OPEN ({self.failure_count} failures)")
        if self.state == "HALF_OPEN":
            self.state = "OPEN"
            self.half_open_successes = 0
    
    def get_status(self):
        retry_in = max(0, self.recovery_timeout - (time.time() - self.last_failure_time)) if self.state == "OPEN" else 0
        return {"state": self.state, "failures": self.failure_count, "retry_in_seconds": round(retry_in)}


# ============================================================
# RAG BACKEND (simplified)
# ============================================================

class RAGBackend:
    def __init__(self):
        self.cache = {}
        self.claude = anthropic.Anthropic()
        self.runbooks = {
            "oom": "OOMKilled Recovery: kubectl top pods. kubectl edit deployment to increase memory. kubectl rollout restart. For batch jobs: use streaming.",
            "crash": "CrashLoopBackOff: kubectl logs --previous. OOMKilled: increase memory. ImagePull: check registry. Rollback: kubectl rollout undo.",
            "latency": "High Latency: kubectl top pods. Check upstream deps. Check recent deploys. Java: check GC pauses.",
            "redis": "Redis: redis-cli ping. Info memory. maxmemory-policy allkeys-lru. Emergency: FLUSHALL.",
            "db": "DB Connections: pg_stat_activity. pg_terminate_backend for idle. pgbouncer for pooling.",
        }
    
    def ask(self, question, model="claude-sonnet-4-5-20250514"):
        cache_key = question.lower().strip()
        if cache_key in self.cache:
            CACHE_OPS.labels(result="hit").inc()
            return {**self.cache[cache_key], "cached": True}
        
        CACHE_OPS.labels(result="miss").inc()
        
        # Retrieve context
        q_lower = question.lower()
        context_parts = [c for k, c in self.runbooks.items() if k in q_lower or any(w in q_lower for w in k.split())]
        context = "\n\n".join(context_parts) if context_parts else "No relevant context found."
        
        response = self.claude.messages.create(
            model=model, max_tokens=1000, temperature=0,
            system="Answer using ONLY the provided context. If insufficient, say 'I don't have information about that.'",
            messages=[{"role": "user", "content": f"<context>\n{context}\n</context>\n\n<question>{question}</question>"}]
        )
        
        answer = response.content[0].text
        no_answer = any(p in answer.lower() for p in ["i don't have", "i don't know", "no information"])
        if no_answer: NO_ANSWER.inc()
        
        result = {"answer": answer, "model": model, "input_tokens": response.usage.input_tokens,
                  "output_tokens": response.usage.output_tokens, "cached": False, "no_answer": no_answer}
        
        if not no_answer:
            self.cache[cache_key] = result
        return result


# ============================================================
# FASTAPI APP
# ============================================================

cost_tracker = CostTracker()
circuit_breaker = CircuitBreaker()
rag = RAGBackend()

@asynccontextmanager
async def lifespan(app):
    logger.info("AI Service starting")
    yield
    logger.info("AI Service stopping")

app = FastAPI(title="SRE AI Platform", version="1.0.0", lifespan=lifespan)

@app.middleware("http")
async def track_requests(request: Request, call_next):
    rid = str(uuid.uuid4())[:8]
    request.state.request_id = rid
    ACTIVE_REQUESTS.inc()
    start = time.time()
    try:
        response = await call_next(request)
        REQUEST_COUNT.labels(endpoint=request.url.path, status=response.status_code).inc()
        REQUEST_LATENCY.labels(endpoint=request.url.path).observe(time.time() - start)
        response.headers["X-Request-ID"] = rid
        return response
    finally:
        ACTIVE_REQUESTS.dec()

class AskRequest(BaseModel):
    question: str
    model: str = "claude-sonnet-4-5-20250514"

@app.post("/ask")
async def ask(req: AskRequest, request: Request):
    rid = getattr(request.state, 'request_id', '?')
    
    ok, reason = cost_tracker.check_budget()
    if not ok:
        raise HTTPException(429, f"Budget exceeded: {reason}")
    
    if not circuit_breaker.can_proceed():
        raise HTTPException(503, "AI service unavailable (circuit breaker open)")
    
    try:
        start = time.time()
        result = rag.ask(req.question, req.model)
        circuit_breaker.record_success()
        cost = cost_tracker.record(req.model, result["input_tokens"], result["output_tokens"]) if not result["cached"] else 0.0
        latency = round(time.time() - start, 2)
        
        logger.info("Query answered", extra={"extra_data": {
            "rid": rid, "model": req.model, "cached": result["cached"],
            "tokens": result["input_tokens"]+result["output_tokens"], "cost": round(cost, 6)}})
        
        return {"answer": result["answer"], "model": req.model, "cached": result["cached"],
                "latency_s": latency, "tokens": result["input_tokens"]+result["output_tokens"],
                "cost_usd": round(cost, 6), "request_id": rid}
    
    except anthropic.APIError as e:
        circuit_breaker.record_failure()
        raise HTTPException(502, f"AI provider error: {e}")

@app.get("/health/live")
async def liveness():
    return {"status": "alive"}

@app.get("/health/ready")
async def readiness():
    cb = circuit_breaker.get_status()
    budget = cost_tracker.get_status()
    ready = cb["state"] != "OPEN" and budget["hourly_remaining"] > 0
    return {"ready": ready, "circuit_breaker": cb["state"], "budget_hourly_remaining": budget["hourly_remaining"]}

@app.get("/metrics")
async def metrics():
    return Response(content=generate_latest(), media_type=CONTENT_TYPE_LATEST)

@app.get("/cost")
async def cost():
    return cost_tracker.get_status()

@app.get("/circuit-breaker")
async def cb():
    return circuit_breaker.get_status()
```

---

### Exercise 2: Docker Packaging (10 min)

**Goal:** Containerize the AI service with Prometheus and Grafana.

Create `week19/Dockerfile`:

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY ai_service.py .
RUN useradd -m appuser
USER appuser
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD curl -f http://localhost:8000/health/live || exit 1
EXPOSE 8000
CMD ["uvicorn", "ai_service:app", "--host", "0.0.0.0", "--port", "8000"]
```

Create `week19/requirements.txt`:

```
fastapi==0.115.0
uvicorn==0.30.0
anthropic==0.40.0
prometheus-client==0.21.0
pydantic==2.9.0
```

Create `week19/docker-compose.yml`:

```yaml
version: '3.8'
services:
  ai-service:
    build: .
    ports: ["8000:8000"]
    environment:
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
  prometheus:
    image: prom/prometheus:latest
    ports: ["9090:9090"]
    volumes: ["./prometheus.yml:/etc/prometheus/prometheus.yml"]
  grafana:
    image: grafana/grafana:latest
    ports: ["3000:3000"]
    environment: ["GF_SECURITY_ADMIN_PASSWORD=admin"]
```

Create `week19/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: 'ai-service'
    static_configs:
      - targets: ['ai-service:8000']
    metrics_path: '/metrics'
```

```bash
docker compose up --build
# AI service: http://localhost:8000
# Prometheus: http://localhost:9090
# Grafana:    http://localhost:3000
```

---

### Exercise 3: Grafana Dashboard & Alerting Rules (15 min)

**Goal:** Define dashboard panels and Prometheus alert rules.

Create `week19/grafana_dashboard.json`:

```json
{
  "dashboard": {
    "title": "SRE AI Platform",
    "panels": [
      {"title": "Request Rate", "type": "graph",
       "targets": [{"expr": "rate(ai_requests_total[5m])", "legendFormat": "{{endpoint}} {{status}}"}]},
      {"title": "Latency P50/P95/P99", "type": "graph",
       "targets": [
         {"expr": "histogram_quantile(0.50, rate(ai_request_seconds_bucket[5m]))", "legendFormat": "P50"},
         {"expr": "histogram_quantile(0.95, rate(ai_request_seconds_bucket[5m]))", "legendFormat": "P95"},
         {"expr": "histogram_quantile(0.99, rate(ai_request_seconds_bucket[5m]))", "legendFormat": "P99"}]},
      {"title": "Token Usage/min", "type": "graph",
       "targets": [{"expr": "rate(ai_tokens_total[1m])*60", "legendFormat": "{{model}} {{direction}}"}]},
      {"title": "Cost $/hour", "type": "stat",
       "targets": [{"expr": "rate(ai_cost_usd_total[1h])*3600"}]},
      {"title": "Cache Hit Rate", "type": "gauge",
       "targets": [{"expr": "rate(ai_cache_ops_total{result='hit'}[5m]) / (rate(ai_cache_ops_total{result='hit'}[5m]) + rate(ai_cache_ops_total{result='miss'}[5m]))"}]},
      {"title": "No-Answer Rate", "type": "graph",
       "targets": [{"expr": "rate(ai_no_answer_total[5m]) / rate(ai_requests_total[5m])"}]}
    ]
  }
}
```

Create `week19/alert_rules.yml`:

```yaml
groups:
- name: ai-platform
  rules:
  - alert: AIHighLatency
    expr: histogram_quantile(0.99, rate(ai_request_seconds_bucket[5m])) > 10
    for: 5m
    labels: {severity: warning}
    annotations: {summary: "P99 latency > 10s"}
  - alert: AIHighCost
    expr: rate(ai_cost_usd_total[1h]) * 3600 > 5
    for: 10m
    labels: {severity: warning}
    annotations: {summary: "Spending > $5/hour"}
  - alert: AIHighNoAnswerRate
    expr: rate(ai_no_answer_total[15m]) / rate(ai_requests_total[15m]) > 0.2
    for: 15m
    labels: {severity: warning}
    annotations: {summary: "20%+ queries getting no answer"}
```

**Dashboard panels explained:**

| Panel | Monitors | Alert If |
|-------|----------|---------|
| Request Rate | Traffic | Sudden spike/drop |
| Latency P99 | User experience | > 10s |
| Token Usage | API consumption | Sudden spike (runaway) |
| Cost/Hour | Budget burn | > $5/hour |
| Cache Hit Rate | Efficiency | < 30% |
| No-Answer Rate | Knowledge gaps | > 20% |

---

## 📝 Drills (20 min)

### Drill 1: Load Test

```python
import requests, time
from concurrent.futures import ThreadPoolExecutor

questions = ["How to fix OOMKilled?", "Redis recovery?", "DB connection issues?",
             "High latency investigation?", "Deployment rollback?"]

def test(q):
    start = time.time()
    r = requests.post("http://localhost:8000/ask", json={"question": q})
    return {"q": q[:25], "ms": round((time.time()-start)*1000), "cached": r.json().get("cached")}

# Sequential
for q in questions:
    print(test(q))

# Parallel burst
with ThreadPoolExecutor(5) as pool:
    for r in pool.map(test, questions * 3):
        print(f"  {r['ms']:>5}ms {'CACHED' if r['cached'] else 'FRESH':>6} {r['q']}")
```

### Drill 2: K8s Deployment Manifest

Write a deployment + service + HPA for the AI service with proper probes, resource limits, and Prometheus annotations.

### Drill 3: Production Readiness Checklist

```
□ Circuit breaker          □ Budget limits
□ Health checks            □ Prometheus metrics
□ Structured logging       □ Request ID tracking
□ Docker packaging         □ Non-root user
□ Grafana dashboard        □ Alert rules
□ Semantic caching         □ Graceful degradation
```

---

## ✅ Week 19 Checklist

- [ ] FastAPI service with /ask, /health, /metrics, /cost, /circuit-breaker
- [ ] Prometheus metrics: latency, tokens, cost, cache, no-answer rate
- [ ] Circuit breaker for upstream API failures
- [ ] Cost tracker with hourly/daily limits
- [ ] Docker + docker-compose (service + Prometheus + Grafana)
- [ ] Grafana dashboard with 6+ panels
- [ ] Alert rules for latency, cost, knowledge gaps

---

## 🧠 Key Concepts

### AI Monitoring Pyramid

```
         /\
        /$$\        Cost & Budget
       /────\
      /Cache \      Efficiency
     / Tokens \
    /──────────\
   / Iterations \   AI Quality
  / No-Answer    \
 /────────────────\
/ Latency │ Errors \  Standard Ops
/──────────────────────\
```

### Interview Angle

"AI services need circuit breakers for upstream LLM outages, budget enforcement to prevent cost spikes, and AI-specific metrics like token usage, cache hit rate, and 'I don't know' rate that signal knowledge base gaps."

---

*Next week: The Capstone! Week 20 brings everything together.*
