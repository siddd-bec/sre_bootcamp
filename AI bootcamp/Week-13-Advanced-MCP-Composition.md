# Week 13: Advanced MCP — Composition & Integration

## 🎯 Learning Objectives

By the end of this week, you will:
- Compose multiple MCP servers into a unified investigation workflow
- Build an MCP client in Python (call MCP servers programmatically, not just via Claude Desktop)
- Integrate RAG retrieval as an automatic context source within MCP tool calls
- Add MCP prompt templates that orchestrate multi-server investigations
- Assemble the final `sre-mcp-server` portfolio project — one server that combines Kubernetes, logs, and runbooks
- Understand MCP best practices for real-world deployment

---

## 📖 Theory (20 min)

### The Multi-Server Problem

In Week 12, you built 3 separate MCP servers. Claude Desktop connects to all three and can call tools from any of them. This works, but has friction:

- Claude must decide WHICH server has the tool it needs
- Tool descriptions across servers might overlap or conflict
- No shared context between servers (Kubernetes data doesn't flow to log queries)
- Managing 3 separate server processes is operational overhead

### Two Solutions

**Option A: Keep separate servers, add smart prompts.**
Use MCP prompt templates that guide Claude to call the RIGHT tools from the RIGHT servers in the RIGHT order. The servers stay independent but the prompts orchestrate them.

**Option B: Build one composite server.**
Combine all tools into a single server. Add "meta-tools" that automatically chain operations (e.g., `investigate_service` calls pods → logs → runbooks internally). Less modular but simpler to operate.

This week we do **both** — learn the patterns, then build the composite server as your portfolio project.

### MCP Client — Calling Servers from Your Code

So far, Claude Desktop has been the MCP host. But you can also build your OWN Python program as an MCP host. This means:
- Your Python scripts can call any MCP server
- You can build automated pipelines that use MCP tools
- Your Week 14-16 agents can use MCP servers as their tool backend

```python
# Your Python code acts as an MCP host
client = MCPClient()
client.connect("kubernetes-sre-server")

# Call tools programmatically
result = await client.call_tool("list_pods", {"namespace": "payments"})
```

### RAG-Augmented Tool Calls

The most powerful pattern: when Claude calls a tool, your server AUTOMATICALLY searches the knowledge base for relevant runbooks and includes them in the response.

```
Claude asks: check_service_health("payment-gateway")

Your server:
  1. Gets health data: {status: "degraded", error: "OOMKilled"}
  2. AUTO-searches RAG: "OOMKilled pod recovery" → finds runbook
  3. Returns BOTH: health data + relevant runbook section

Claude gets health data AND the fix procedure in one tool call!
```

This eliminates the need for Claude to make a separate runbook search call — it gets context automatically.

---

## 🔨 Hands-On (50 min)

### Exercise 1: MCP Prompt Templates for Orchestration (10 min)

**Goal:** Create prompt templates that guide Claude through a structured multi-tool investigation.

Create `week13/investigation_prompts.py` — these prompts get added to any of your MCP servers:

```python
"""
Week 13, Exercise 1: MCP Prompt Templates for Orchestration

Prompts guide Claude through structured investigations,
ensuring it calls the RIGHT tools in the RIGHT order.
"""
from mcp.types import Prompt, PromptArgument, PromptMessage, TextContent, GetPromptResult

# ============================================================
# PROMPT DEFINITIONS
# ============================================================

PROMPTS = {
    "investigate-service": {
        "description": "Full service investigation: check health, pods, logs, deployments, and find relevant runbooks",
        "arguments": [
            PromptArgument(name="service", description="Service name to investigate", required=True),
            PromptArgument(name="symptoms", description="What symptoms are observed", required=False),
        ],
        "template": """Investigate the {service} service systematically. {symptoms_line}

Follow this investigation procedure IN ORDER:

**Step 1 — Health Check**
Use check_service_health to get current status, error rate, and latency.

**Step 2 — Pod Status**
Use list_pods to see which pods are running, crashing, or pending.
If any pods are unhealthy, use describe_pod on the worst one.

**Step 3 — Logs**
Use get_pod_logs on any unhealthy pods (with severity=ERROR).
Also check WARN logs for early warning signs.

**Step 4 — Deployment History**
Use get_recent_deploys to check if a recent deployment caused this.

**Step 5 — Log Analysis**
Use search_logs to find error patterns across the service.
Use count_errors to see the distribution.

**Step 6 — Knowledge Base**
Use search_runbooks to find any relevant runbooks for the issues found.

**Step 7 — Synthesis**
After gathering ALL data, provide:
1. ROOT CAUSE: What is causing the issue (based on evidence)
2. SEVERITY: P1/P2/P3/P4
3. IMMEDIATE ACTIONS: Specific commands to run NOW
4. RUNBOOK REFERENCE: Which runbook to follow
5. FOLLOW-UP: What to monitor after fixing"""
    },
    
    "trace-transaction": {
        "description": "Trace a specific transaction across all services to find where it failed",
        "arguments": [
            PromptArgument(name="trace_id", description="Transaction trace ID", required=True),
        ],
        "template": """Trace transaction {trace_id} across all services.

**Step 1:** Use trace_transaction to get the full transaction lifecycle.
**Step 2:** For each service that shows an error, use get_pod_logs to get more detail.
**Step 3:** Use search_runbooks to find fixes for any errors discovered.
**Step 4:** Provide a timeline of what happened and where the failure occurred."""
    },
    
    "pre-deploy-check": {
        "description": "Pre-deployment health check — verify everything is healthy before deploying",
        "arguments": [
            PromptArgument(name="service", description="Service about to be deployed", required=True),
            PromptArgument(name="namespace", description="Kubernetes namespace", required=True),
        ],
        "template": """Run a pre-deployment health check for {service} in namespace {namespace}.

Check ALL of the following before giving a GO/NO-GO:

1. **Pod Health:** list_pods — all pods Running? Any recent restarts?
2. **Error Rate:** search_logs for recent errors in the service
3. **Dependencies:** check_service_health on upstream services
4. **Recent Events:** get_events — any warnings?
5. **Slow Queries:** get_slow_queries — any database pressure?

Provide a clear **GO** or **NO-GO** recommendation with evidence."""
    },

    "daily-health-report": {
        "description": "Generate a daily health report across all services",
        "arguments": [],
        "template": """Generate a daily SRE health report.

For each service (payment-gateway, auth-service, fraud-detection, notification-service):
1. Use check_service_health to get current status
2. Use count_errors grouped by service

Then provide:
- Overall system health: GREEN / YELLOW / RED
- Services with issues (if any)
- Action items for the on-call team
- Notable trends (error rates trending up/down)

Keep the report concise — it should fit in a Slack message."""
    }
}


def get_prompt_definitions() -> list[Prompt]:
    """Return all prompt definitions for MCP registration."""
    prompts = []
    for name, config in PROMPTS.items():
        prompts.append(Prompt(
            name=name,
            description=config["description"],
            arguments=config.get("arguments", [])
        ))
    return prompts


def render_prompt(name: str, arguments: dict) -> GetPromptResult:
    """Render a prompt template with provided arguments."""
    if name not in PROMPTS:
        raise ValueError(f"Unknown prompt: {name}")
    
    config = PROMPTS[name]
    template = config["template"]
    
    # Handle optional arguments
    if name == "investigate-service":
        symptoms = arguments.get("symptoms", "")
        symptoms_line = f"Reported symptoms: {symptoms}" if symptoms else ""
        text = template.format(
            service=arguments["service"],
            symptoms_line=symptoms_line
        )
    elif name == "trace-transaction":
        text = template.format(trace_id=arguments["trace_id"])
    elif name == "pre-deploy-check":
        text = template.format(
            service=arguments["service"],
            namespace=arguments["namespace"]
        )
    else:
        text = template
    
    return GetPromptResult(
        messages=[
            PromptMessage(role="user", content=TextContent(type="text", text=text))
        ]
    )
```

---

### Exercise 2: RAG-Augmented Tool Responses (15 min)

**Goal:** When a tool detects an issue, automatically search the knowledge base and include relevant runbook context in the response.

Create `week13/rag_augmented_tools.py`:

```python
"""
Week 13, Exercise 2: RAG-Augmented Tool Responses

When a tool finds a problem, automatically search for the relevant runbook
and include it in the response. Claude gets both DATA and FIX in one call.
"""
import json
import chromadb
from sentence_transformers import SentenceTransformer

embedder = SentenceTransformer('all-MiniLM-L6-v2')


class RAGAugmenter:
    """
    Automatically augments tool responses with relevant runbook context.
    
    Usage:
        augmenter = RAGAugmenter()
        augmenter.load_runbooks(runbooks_dict)
        
        # In your tool handler:
        raw_result = check_pods(...)
        augmented = augmenter.augment(raw_result)
        # augmented now includes relevant_runbooks field
    """
    
    def __init__(self):
        self.chroma = chromadb.Client()
        self.collection = self.chroma.get_or_create_collection("rag_augment")
        self.loaded = False
    
    def load_runbooks(self, runbooks: dict):
        """Load runbooks into the search index."""
        for key, rb in runbooks.items():
            emb = embedder.encode(f"{rb['title']} {rb['content']}").tolist()
            try:
                self.collection.add(
                    ids=[key],
                    documents=[rb['content']],
                    metadatas=[{"title": rb['title']}],
                    embeddings=[emb]
                )
            except Exception:
                pass
        self.loaded = True
    
    def find_relevant_runbooks(self, context: str, n_results: int = 2, threshold: float = 1.5) -> list[dict]:
        """Search for runbooks relevant to the given context."""
        if not self.loaded or not context:
            return []
        
        query_emb = embedder.encode(context).tolist()
        results = self.collection.query(
            query_embeddings=[query_emb],
            n_results=n_results,
            include=["documents", "metadatas", "distances"]
        )
        
        relevant = []
        for i in range(len(results['ids'][0])):
            if results['distances'][0][i] <= threshold:
                relevant.append({
                    "title": results['metadatas'][0][i]['title'],
                    "content": results['documents'][0][i],
                    "relevance": round(1 - results['distances'][0][i] / 2, 3)
                })
        
        return relevant
    
    def augment(self, tool_result: dict, tool_name: str = "") -> dict:
        """
        Augment a tool result with relevant runbook context.
        
        Automatically detects issues in the result and searches for fixes.
        """
        # Extract signals that indicate problems
        issue_signals = []
        
        result_str = json.dumps(tool_result).lower()
        
        # Detect common issues from tool output
        issue_patterns = {
            "oomkilled": "pod OOMKilled out of memory recovery",
            "crashloopbackoff": "pod CrashLoopBackOff troubleshooting",
            "connection refused": "connection refused service unreachable",
            "connection pool": "database connection pool exhaustion",
            "timeout": "request timeout latency investigation",
            "high latency": "service high latency troubleshooting",
            "circuit breaker": "circuit breaker open failure",
            "disk pressure": "disk space full cleanup",
            "memory pressure": "node memory pressure eviction",
            "redis": "Redis cache failure recovery",
            "outofmemoryerror": "Java OutOfMemoryError heap space",
            "full gc": "Java garbage collection performance",
        }
        
        for pattern, search_query in issue_patterns.items():
            if pattern in result_str:
                issue_signals.append(search_query)
        
        if not issue_signals:
            return tool_result  # No issues detected — return as-is
        
        # Search for runbooks matching the detected issues
        combined_query = " ".join(issue_signals[:3])  # Top 3 signals
        runbooks = self.find_relevant_runbooks(combined_query)
        
        if runbooks:
            tool_result["_relevant_runbooks"] = runbooks
            tool_result["_augmentation_note"] = (
                f"Detected issue signals: {issue_signals[:3]}. "
                f"Found {len(runbooks)} relevant runbook(s)."
            )
        
        return tool_result


# ============================================================
# DEMO: Augmented tool call
# ============================================================

RUNBOOKS = {
    "pod-crash": {
        "title": "Pod CrashLoopBackOff Recovery",
        "content": "OOMKilled: increase memory limits with kubectl edit deployment. Check current: kubectl top pods. Rollback: kubectl rollout undo."
    },
    "redis-fail": {
        "title": "Redis Cache Recovery",
        "content": "Redis down: redis-cli ping. Check memory: redis-cli info memory. Eviction: maxmemory-policy allkeys-lru. Emergency: FLUSHALL."
    },
    "db-conn": {
        "title": "PostgreSQL Connection Issues",
        "content": "Connections exhausted: SELECT count(*) FROM pg_stat_activity. Kill idle: pg_terminate_backend(pid). Long-term: pgbouncer."
    },
    "latency": {
        "title": "High Latency Investigation",
        "content": "Check resources: kubectl top pods. Check dependencies. Check recent deployments. Rollback: kubectl rollout undo."
    },
}

if __name__ == "__main__":
    augmenter = RAGAugmenter()
    augmenter.load_runbooks(RUNBOOKS)
    
    # Simulate a tool result that shows OOMKilled pods
    tool_result = {
        "namespace": "payments",
        "pods": [
            {"name": "payment-gw-def34", "status": "CrashLoopBackOff", "restarts": 47},
            {"name": "payment-gw-ghi56", "status": "CrashLoopBackOff", "restarts": 32},
        ],
        "events": [
            {"reason": "OOMKilled", "message": "Container exceeded memory limit 512Mi"}
        ]
    }
    
    print("=" * 70)
    print("BEFORE augmentation:")
    print("=" * 70)
    print(json.dumps(tool_result, indent=2))
    
    augmented = augmenter.augment(tool_result)
    
    print(f"\n{'=' * 70}")
    print("AFTER augmentation:")
    print("=" * 70)
    print(json.dumps(augmented, indent=2))
    
    print(f"""
\n💡 The tool result now includes:
  - Original data (pod status, events)
  - Automatically found runbooks for OOMKilled issues
  - Claude gets the problem AND the fix in one response
  
  Without augmentation: Claude needs 2 tool calls (check pods + search runbooks)
  With augmentation: Claude gets everything in 1 tool call
""")
```

---

### Exercise 3: Composite SRE MCP Server — Portfolio Project (25 min)

**Goal:** Build the final `sre-mcp-server` — a single production server combining Kubernetes, ClickHouse logs, and RAG-augmented runbooks.

Create `week13/sre_mcp_server.py`:

```python
"""
Week 13, Exercise 3: Composite SRE MCP Server

Your portfolio project: ONE server that combines:
- Kubernetes cluster tools (5 tools)
- ClickHouse log query tools (4 tools)
- RAG runbook search (1 tool)
- RAG-augmented responses (automatic)
- Investigation prompt templates (4 prompts)
- Browsable runbook resources (5 resources)

Total: 10 tools, 4 prompts, 5 resources
"""
import json
import logging
import re
import chromadb
from sentence_transformers import SentenceTransformer
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent, Resource, Prompt, PromptArgument, PromptMessage, GetPromptResult

# ============================================================
# SETUP
# ============================================================

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s | %(levelname)s | %(name)s | %(message)s',
    handlers=[logging.FileHandler("sre_mcp_server.log"), logging.StreamHandler()]
)
logger = logging.getLogger("sre-mcp")

server = Server("sre-mcp-server")
embedder = SentenceTransformer('all-MiniLM-L6-v2')

# Vector DB for runbooks
chroma = chromadb.Client()
runbook_collection = chroma.get_or_create_collection("sre_runbooks")

ALLOWED_NAMESPACES = ["payments", "auth", "monitoring", "default"]


# ============================================================
# RUNBOOK DATA + LOADING
# ============================================================

RUNBOOKS = {
    "pod-crash": {"title": "Pod CrashLoopBackOff Recovery", "team": "platform",
        "content": "When pod is CrashLoopBackOff: kubectl logs --previous for crash reason. OOMKilled: increase memory limits (kubectl edit deployment, change resources.limits.memory). Image pull error: check registry credentials. App crash: check stack traces. Liveness probe: increase initialDelaySeconds. Quick fix: kubectl rollout restart. Rollback: kubectl rollout undo."},
    "redis-fail": {"title": "Redis Cache Recovery", "team": "data",
        "content": "Redis down: redis-cli ping (expect PONG). Memory: redis-cli info memory. If full: CONFIG SET maxmemory 4gb, check maxmemory-policy (allkeys-lru). Clients: redis-cli info clients. Cluster: redis-cli cluster info. Emergency: FLUSHALL. Monitor hit rate recovery to >90%."},
    "db-conn": {"title": "PostgreSQL Connection Issues", "team": "data",
        "content": "Connections exhausted: SELECT count(*), state FROM pg_stat_activity GROUP BY state. Kill idle: pg_terminate_backend(pid) WHERE state='idle' AND query_start < now()-interval '10 min'. SHOW max_connections. Long-term: pgbouncer. Pool sizing: max_connections / pods * 0.8."},
    "latency": {"title": "High Latency Investigation", "team": "platform",
        "content": "Scope: all endpoints or specific? Check resources: kubectl top pods/nodes. Check upstream: database, Redis, external APIs. Check deploys: kubectl rollout history. Traffic anomalies: compare QPS to baseline. Java: check GC pauses, take thread/heap dump."},
    "disk-space": {"title": "Disk Space Emergency", "team": "platform",
        "content": "Disk full: du -sh /* to identify. Quick cleanup: journalctl --vacuum-time=3d, docker system prune, find /tmp -mtime +7 -delete. K8s: crictl rmi --prune. Log rotation: logrotate -f /etc/logrotate.conf. Long-term: PVC auto-expansion, alert at 80%."},
}

def _load_runbooks():
    for key, rb in RUNBOOKS.items():
        emb = embedder.encode(f"{rb['title']} {rb['content']}").tolist()
        try:
            runbook_collection.add(ids=[key], documents=[rb['content']],
                metadatas=[{"title": rb['title'], "team": rb['team']}], embeddings=[emb])
        except Exception:
            pass

_load_runbooks()


# ============================================================
# RAG AUGMENTER
# ============================================================

ISSUE_PATTERNS = {
    "oomkilled": "pod OOMKilled memory recovery",
    "crashloopbackoff": "CrashLoopBackOff troubleshooting",
    "connection refused": "connection refused unreachable",
    "timeout": "request timeout latency",
    "circuit breaker": "circuit breaker failure",
    "outofmemoryerror": "Java OutOfMemoryError heap",
    "disk pressure": "disk space full",
    "redis": "Redis cache failure",
    "connection pool": "database connection pool",
    "full gc": "Java GC performance",
}

def auto_augment(result: dict) -> dict:
    """Detect issues in tool output and attach relevant runbooks."""
    result_lower = json.dumps(result).lower()
    signals = [q for pattern, q in ISSUE_PATTERNS.items() if pattern in result_lower]
    
    if not signals:
        return result
    
    query = " ".join(signals[:3])
    emb = embedder.encode(query).tolist()
    search = runbook_collection.query(query_embeddings=[emb], n_results=2,
        include=["documents", "metadatas", "distances"])
    
    runbooks = []
    for i in range(len(search['ids'][0])):
        if search['distances'][0][i] < 1.5:
            runbooks.append({
                "title": search['metadatas'][0][i]['title'],
                "content": search['documents'][0][i][:300] + "...",
            })
    
    if runbooks:
        result["_relevant_runbooks"] = runbooks
    
    return result


# ============================================================
# SIMULATED DATA (same as Week 12, condensed)
# ============================================================

K8S = {
    "payments": {
        "pods": [
            {"name": "payment-gw-abc12", "status": "Running", "ready": "1/1", "restarts": 0, "node": "worker-01"},
            {"name": "payment-gw-def34", "status": "CrashLoopBackOff", "ready": "0/1", "restarts": 47, "node": "worker-02"},
            {"name": "payment-gw-ghi56", "status": "CrashLoopBackOff", "ready": "0/1", "restarts": 32, "node": "worker-01"},
            {"name": "payment-gw-jkl78", "status": "Running", "ready": "1/1", "restarts": 0, "node": "worker-03"},
            {"name": "payment-gw-mno90", "status": "Running", "ready": "1/1", "restarts": 0, "node": "worker-02"},
        ],
        "deployments": [{"name": "payment-gateway", "ready": "3/5", "image": "visa/payment-gw:v2.3.1", "age": "45d"}],
        "events": [
            {"time": "14:23", "type": "Warning", "reason": "OOMKilled", "object": "pod/payment-gw-def34", "message": "Container exceeded memory limit (512Mi)"},
            {"time": "14:22", "type": "Warning", "reason": "BackOff", "object": "pod/payment-gw-def34", "message": "Back-off restarting failed container"},
            {"time": "14:00", "type": "Normal", "reason": "ScalingReplicaSet", "object": "deployment/payment-gateway", "message": "Scaled up to 5"},
        ],
        "pod_logs": {
            "payment-gw-def34": [
                "14:23:05 INFO  [BatchReconciliation] Starting daily reconciliation",
                "14:23:06 WARN  [BatchReconciliation] Loading 2.1M transactions into memory",
                "14:23:08 WARN  [GC] Full GC — heap at 97%",
                "14:23:11 ERROR [BatchReconciliation] java.lang.OutOfMemoryError: Java heap space",
                "14:23:11 ERROR   at BatchReconciliation.loadAllTransactions(BatchReconciliation.java:142)",
            ],
        },
        "pod_describe": {
            "payment-gw-def34": {
                "status": "CrashLoopBackOff", "last_state": "OOMKilled (exit 137)",
                "resources": {"requests": {"memory": "256Mi"}, "limits": {"memory": "512Mi"}},
                "restart_count": 47,
            }
        }
    }
}

SERVICES = {
    "payment-gateway": {"status": "degraded", "error_rate": 4.7, "p99_ms": 2100, "pods": "3/5", "alerts": ["ErrorRateHigh", "PodCrashLoopBackOff"]},
    "auth-service": {"status": "healthy", "error_rate": 0.01, "p99_ms": 42, "pods": "4/4", "alerts": []},
    "fraud-detection": {"status": "healthy", "error_rate": 0.05, "p99_ms": 230, "pods": "6/6", "alerts": []},
    "notification-service": {"status": "healthy", "error_rate": 0.02, "p99_ms": 55, "pods": "3/3", "alerts": []},
}

LOGS = [
    {"ts": "14:23:01", "svc": "payment-gateway", "level": "ERROR", "msg": "OutOfMemoryError: Java heap space", "trace": "tx-abc123"},
    {"ts": "14:23:02", "svc": "payment-gateway", "level": "ERROR", "msg": "Transaction tx-def456 failed: timeout 30s", "trace": "tx-def456"},
    {"ts": "14:23:03", "svc": "payment-gateway", "level": "ERROR", "msg": "Circuit breaker OPEN for card-network-gateway", "trace": ""},
    {"ts": "14:22:30", "svc": "fraud-detection", "level": "WARN", "msg": "Model inference latency 890ms (threshold 500ms)", "trace": "tx-abc123"},
    {"ts": "14:23:01", "svc": "fraud-detection", "level": "ERROR", "msg": "Redis cache miss for velocity data", "trace": "tx-abc123"},
    {"ts": "14:22:00", "svc": "auth-service", "level": "INFO", "msg": "Health check passed", "trace": ""},
]

DEPLOYS = {
    "payment-gateway": [
        {"version": "v2.3.1", "time": "14:00 today", "changes": "Added BatchReconciliation for daily reconciliation"},
        {"version": "v2.3.0", "time": "yesterday 10:00", "changes": "Bug fix for card network timeout"},
    ],
}


# ============================================================
# TOOL IMPLEMENTATIONS (10 tools)
# ============================================================

def _validate_ns(ns): 
    ns = ns.strip().lower()
    if ns not in ALLOWED_NAMESPACES: raise ValueError(f"Namespace '{ns}' not allowed")
    return ns

def check_service_health(service_name):
    return SERVICES.get(service_name, {"error": f"Unknown: {service_name}", "available": list(SERVICES.keys())})

def list_pods(namespace, status_filter=None):
    ns = _validate_ns(namespace)
    pods = K8S.get(ns, {}).get("pods", [])
    if status_filter: pods = [p for p in pods if p["status"].lower() == status_filter.lower()]
    return {"namespace": ns, "pods": pods, "count": len(pods)}

def get_pod_logs(namespace, pod_name, lines=20):
    ns = _validate_ns(namespace)
    logs = K8S.get(ns, {}).get("pod_logs", {}).get(pod_name, [])
    return {"pod": pod_name, "logs": logs[-lines:]}

def describe_pod(namespace, pod_name):
    ns = _validate_ns(namespace)
    return K8S.get(ns, {}).get("pod_describe", {}).get(pod_name, {"error": "Pod not found"})

def list_deployments(namespace):
    ns = _validate_ns(namespace)
    return {"namespace": ns, "deployments": K8S.get(ns, {}).get("deployments", [])}

def get_events(namespace, event_type=None):
    ns = _validate_ns(namespace)
    events = K8S.get(ns, {}).get("events", [])
    if event_type: events = [e for e in events if e["type"].lower() == event_type.lower()]
    return {"namespace": ns, "events": events}

def search_logs(service=None, level=None, keyword=None, limit=20):
    results = LOGS.copy()
    if service: results = [r for r in results if r["svc"] == service]
    if level: results = [r for r in results if r["level"] == level.upper()]
    if keyword: results = [r for r in results if keyword.lower() in r["msg"].lower()]
    return {"logs": results[:min(limit, 100)], "count": len(results)}

def count_errors(group_by="service"):
    errors = [r for r in LOGS if r["level"] == "ERROR"]
    counts = {}
    for e in errors:
        key = e["svc"] if group_by == "service" else e["msg"][:40]
        counts[key] = counts.get(key, 0) + 1
    return {"groups": [{"key": k, "count": v} for k, v in sorted(counts.items(), key=lambda x: -x[1])]}

def get_recent_deploys(service_name, hours_back=24):
    return {"service": service_name, "deployments": DEPLOYS.get(service_name, [])}

def search_runbooks(query, n_results=3, team_filter=None):
    emb = embedder.encode(query).tolist()
    where = {"team": team_filter} if team_filter else None
    results = runbook_collection.query(query_embeddings=[emb], n_results=n_results, where=where,
        include=["documents", "metadatas", "distances"])
    formatted = []
    for i in range(len(results['ids'][0])):
        formatted.append({"title": results['metadatas'][0][i]['title'],
            "content": results['documents'][0][i], "relevance": round(1 - results['distances'][0][i]/2, 3)})
    return {"results": formatted}


# ============================================================
# MCP REGISTRATION — TOOLS
# ============================================================

TOOL_DEFS = [
    Tool(name="check_service_health", description="Check production service health: status, error rate, latency, active alerts. First step for any investigation.",
        inputSchema={"type": "object", "properties": {"service_name": {"type": "string", "enum": list(SERVICES.keys())}}, "required": ["service_name"]}),
    Tool(name="list_pods", description="List Kubernetes pods in a namespace with status and restart counts.",
        inputSchema={"type": "object", "properties": {"namespace": {"type": "string"}, "status_filter": {"type": "string"}}, "required": ["namespace"]}),
    Tool(name="get_pod_logs", description="Get recent log lines from a pod. Use for error messages and stack traces.",
        inputSchema={"type": "object", "properties": {"namespace": {"type": "string"}, "pod_name": {"type": "string"}, "lines": {"type": "integer", "default": 20}}, "required": ["namespace", "pod_name"]}),
    Tool(name="describe_pod", description="Get detailed pod info: resources, probes, conditions, last termination reason.",
        inputSchema={"type": "object", "properties": {"namespace": {"type": "string"}, "pod_name": {"type": "string"}}, "required": ["namespace", "pod_name"]}),
    Tool(name="list_deployments", description="List deployments with replica counts and image versions.",
        inputSchema={"type": "object", "properties": {"namespace": {"type": "string"}}, "required": ["namespace"]}),
    Tool(name="get_events", description="Get recent Kubernetes events (warnings, errors, scaling).",
        inputSchema={"type": "object", "properties": {"namespace": {"type": "string"}, "event_type": {"type": "string", "enum": ["Warning", "Normal"]}}, "required": ["namespace"]}),
    Tool(name="search_logs", description="Search application logs in ClickHouse by service, level, keyword.",
        inputSchema={"type": "object", "properties": {"service": {"type": "string"}, "level": {"type": "string", "enum": ["ERROR", "WARN", "INFO"]}, "keyword": {"type": "string"}, "limit": {"type": "integer", "default": 20}}}),
    Tool(name="count_errors", description="Count errors grouped by service or message type.",
        inputSchema={"type": "object", "properties": {"group_by": {"type": "string", "enum": ["service", "message"], "default": "service"}}}),
    Tool(name="get_recent_deploys", description="Get deployment history for a service in the last 24 hours.",
        inputSchema={"type": "object", "properties": {"service_name": {"type": "string"}}, "required": ["service_name"]}),
    Tool(name="search_runbooks", description="Search SRE runbooks using natural language. Finds relevant troubleshooting procedures.",
        inputSchema={"type": "object", "properties": {"query": {"type": "string"}, "n_results": {"type": "integer", "default": 3}, "team_filter": {"type": "string"}}, "required": ["query"]}),
]

TOOL_HANDLERS = {
    "check_service_health": lambda a: auto_augment(check_service_health(a["service_name"])),
    "list_pods": lambda a: auto_augment(list_pods(a["namespace"], a.get("status_filter"))),
    "get_pod_logs": lambda a: get_pod_logs(a["namespace"], a["pod_name"], a.get("lines", 20)),
    "describe_pod": lambda a: auto_augment(describe_pod(a["namespace"], a["pod_name"])),
    "list_deployments": lambda a: list_deployments(a["namespace"]),
    "get_events": lambda a: get_events(a["namespace"], a.get("event_type")),
    "search_logs": lambda a: search_logs(a.get("service"), a.get("level"), a.get("keyword"), a.get("limit", 20)),
    "count_errors": lambda a: count_errors(a.get("group_by", "service")),
    "get_recent_deploys": lambda a: get_recent_deploys(a["service_name"], a.get("hours_back", 24)),
    "search_runbooks": lambda a: search_runbooks(a["query"], a.get("n_results", 3), a.get("team_filter")),
}

@server.list_tools()
async def handle_list_tools() -> list[Tool]:
    return TOOL_DEFS

@server.call_tool()
async def handle_call_tool(name: str, arguments: dict) -> list[TextContent]:
    logger.info(f"TOOL | {name} | {json.dumps(arguments)}")
    try:
        handler = TOOL_HANDLERS.get(name)
        if not handler:
            return [TextContent(type="text", text=json.dumps({"error": f"Unknown tool: {name}"}))]
        result = handler(arguments)
        return [TextContent(type="text", text=json.dumps(result, indent=2))]
    except ValueError as e:
        return [TextContent(type="text", text=json.dumps({"error": str(e)}))]
    except Exception as e:
        logger.error(f"ERROR | {name} | {e}")
        return [TextContent(type="text", text=json.dumps({"error": f"Internal error: {str(e)}"}))]


# ============================================================
# MCP REGISTRATION — RESOURCES
# ============================================================

@server.list_resources()
async def handle_list_resources() -> list[Resource]:
    return [Resource(uri=f"runbook://{k}", name=rb["title"],
        description=f"SRE runbook: {rb['title']}", mimeType="text/plain")
        for k, rb in RUNBOOKS.items()]

@server.read_resource()
async def handle_read_resource(uri: str) -> str:
    key = uri.replace("runbook://", "")
    if key in RUNBOOKS:
        rb = RUNBOOKS[key]
        return f"# {rb['title']}\nTeam: {rb['team']}\n\n{rb['content']}"
    raise ValueError(f"Not found: {key}")


# ============================================================
# MCP REGISTRATION — PROMPTS
# ============================================================

@server.list_prompts()
async def handle_list_prompts() -> list[Prompt]:
    return [
        Prompt(name="investigate-service", description="Full service investigation using all available tools",
            arguments=[PromptArgument(name="service", description="Service to investigate", required=True),
                       PromptArgument(name="symptoms", description="Observed symptoms", required=False)]),
        Prompt(name="pre-deploy-check", description="Pre-deployment health verification",
            arguments=[PromptArgument(name="service", description="Service to deploy", required=True)]),
        Prompt(name="daily-health-report", description="Daily health report across all services", arguments=[]),
    ]

@server.get_prompt()
async def handle_get_prompt(name: str, arguments: dict) -> GetPromptResult:
    if name == "investigate-service":
        svc = arguments.get("service", "unknown")
        symptoms = arguments.get("symptoms", "")
        text = f"""Investigate {svc} systematically. {"Symptoms: " + symptoms if symptoms else ""}

Steps: 1) check_service_health 2) list_pods (payments namespace) 3) get_pod_logs on unhealthy pods 4) get_events for warnings 5) search_logs for errors 6) get_recent_deploys 7) search_runbooks for fixes.

After gathering ALL data, provide: ROOT CAUSE, SEVERITY, IMMEDIATE ACTIONS with commands, RUNBOOK REFERENCE."""
    elif name == "pre-deploy-check":
        svc = arguments.get("service", "unknown")
        text = f"Pre-deploy check for {svc}: 1) check_service_health 2) list_pods 3) count_errors 4) search_logs for recent errors. Give GO/NO-GO."
    elif name == "daily-health-report":
        text = "Generate daily health report: check_service_health for each service, count_errors. Format: overall GREEN/YELLOW/RED, issues, action items."
    else:
        raise ValueError(f"Unknown prompt: {name}")
    
    return GetPromptResult(messages=[PromptMessage(role="user", content=TextContent(type="text", text=text))])


# ============================================================
# RUN
# ============================================================

async def main():
    logger.info(f"SRE MCP Server starting — {len(TOOL_DEFS)} tools, {len(RUNBOOKS)} runbooks, 3 prompts")
    async with stdio_server() as (read, write):
        await server.run(read, write, server.create_initialization_options())

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

**Claude Desktop config (just ONE server now):**

```json
{
  "mcpServers": {
    "sre": {
      "command": "python",
      "args": ["/path/to/week13/sre_mcp_server.py"]
    }
  }
}
```

---

## 📝 Drills (20 min)

### Drill 1: MCP Client in Python

Build a simple MCP client that calls your server programmatically:

```python
"""Call MCP server tools from Python (without Claude Desktop)."""
import asyncio
from mcp.client import ClientSession
from mcp.client.stdio import stdio_client, StdioServerParameters

async def main():
    params = StdioServerParameters(
        command="python",
        args=["sre_mcp_server.py"]
    )
    
    async with stdio_client(params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            
            # List available tools
            tools = await session.list_tools()
            print(f"Available tools: {[t.name for t in tools.tools]}")
            
            # Call a tool
            result = await session.call_tool("check_service_health", 
                                              {"service_name": "payment-gateway"})
            print(f"Result: {result}")

asyncio.run(main())
```

This is how your Week 14-16 agents will use MCP servers as their tool backend.

### Drill 2: Add a PagerDuty Integration Tool

Add a tool that creates PagerDuty incidents:

```python
Tool(
    name="create_pagerduty_incident",
    description="Create a PagerDuty incident for the on-call team. Use when investigation reveals a P1 or P2 issue that needs immediate human attention.",
    inputSchema={...}
)
```

Simulate the response but structure it like a real PagerDuty API response.

### Drill 3: Performance Profiling

Measure the latency of each tool call:

```python
import time

@server.call_tool()
async def handle_call_tool(name, arguments):
    start = time.time()
    result = TOOL_HANDLERS[name](arguments)
    latency_ms = (time.time() - start) * 1000
    
    result["_latency_ms"] = round(latency_ms, 1)
    logger.info(f"PERF | {name} | {latency_ms:.1f}ms")
```

Which tools are slowest? How does RAG augmentation affect latency?

### Drill 4: Write the Portfolio README

Create `week13/README.md`:

```markdown
# SRE MCP Server

Production MCP server giving Claude access to Kubernetes, ClickHouse logs,
and SRE runbooks — all in one server.

## Features
- 10 tools for SRE investigation
- RAG-augmented responses (auto-attaches relevant runbooks)
- 5 browsable runbook resources
- 3 investigation prompt templates
- Input validation, audit logging, error handling

## Quick Start
1. pip install mcp sentence-transformers chromadb
2. Add to Claude Desktop config (see below)
3. Ask Claude: "Investigate the payment gateway"
```

---

## ✅ Week 13 Checklist

Before moving to Week 14:

- [ ] Have prompt templates that orchestrate multi-tool investigations
- [ ] Understand RAG-augmented tool responses (auto-attach runbooks)
- [ ] Built the composite `sre-mcp-server` with 10 tools + resources + prompts
- [ ] Can call MCP servers from Python (MCP client pattern)
- [ ] Server has audit logging, input validation, and error handling
- [ ] Have a portfolio README for the sre-mcp-server project

---

## 🧠 Key Concepts from This Week

### The Composite Server Pattern

```
BEFORE (3 servers):                  AFTER (1 server):
  kubernetes-sre  ←MCP→ Claude        sre-mcp-server ←MCP→ Claude
  clickhouse-logs ←MCP→ Claude          ├── K8s tools (5)
  sre-kb          ←MCP→ Claude          ├── Log tools (4)
                                        ├── RAG tool (1)
                                        ├── Auto-augmentation
                                        ├── Prompts (3)
                                        └── Resources (5)
```

### RAG Augmentation Flow

```
Claude calls: list_pods("payments")
  ↓
Server executes: gets pod data
  ↓
Auto-augment detects: "CrashLoopBackOff", "OOMKilled"
  ↓
RAG search: finds Pod Recovery runbook
  ↓
Returns: pod data + runbook section
  ↓
Claude: gets the problem AND the fix in ONE tool call
```

---

## 🏆 Phase 4 Complete!

You've built a complete MCP system:
- Week 11: MCP concepts, first server, Claude Desktop integration
- Week 12: Production servers for K8s, ClickHouse, RAG
- Week 13: Composition, augmentation, composite portfolio server

**Portfolio project:** `sre-mcp-server` — 10 tools, 5 resources, 3 prompts, RAG augmentation.

---

*Next week: Agentic AI — ReAct pattern agents that reason, plan, and use your MCP tools autonomously. This is where everything comes together.*
