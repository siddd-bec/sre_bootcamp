# Week 12: Building Custom MCP Servers

## 🎯 Learning Objectives

By the end of this week, you will:
- Build a production-quality MCP server with multiple tools, resources, and error handling
- Create a Kubernetes MCP server that gives Claude real cluster access
- Create a ClickHouse MCP server for log querying (directly relevant to your Visa migration)
- Expose your RAG knowledge base as an MCP resource + tool
- Handle authentication, input validation, and security in MCP servers
- Understand how to package and distribute MCP servers

---

## 📖 Theory (20 min)

### From Toy Server to Production Server

Your Week 11 server worked, but production MCP servers need:

| Concern | Week 11 (Toy) | Week 12 (Production) |
|---------|--------------|---------------------|
| Error handling | None — crashes on error | Try/catch every tool, structured error returns |
| Input validation | Trust everything | Validate all arguments before executing |
| Security | No auth | API keys, restricted operations, audit logging |
| Observability | print() | Structured logging with timestamps and context |
| Configuration | Hardcoded values | Environment variables, config files |
| Testing | Manual testing | Automated test suite |

### MCP Server Design Patterns

**Pattern 1: Domain Server**
One server per domain — all tools related to one system.
```
kubernetes-server: get_pods, get_logs, describe_pod, scale_deployment
database-server:   query, list_tables, explain_query, check_health
```

**Pattern 2: Capability Server**
One server per capability — tools grouped by what they DO.
```
monitor-server:    check_health, get_metrics, list_alerts
deploy-server:     list_deploys, rollback, scale
investigate-server: search_logs, get_traces, correlate_events
```

**Pattern 3: Composite Server**
One big server with everything — simpler to manage, but less modular.
```
sre-server: all SRE tools in one place
```

For your portfolio, we'll build **domain servers** (Pattern 1) — they're the most reusable and align with how the MCP ecosystem works.

### Security Considerations

MCP servers can do powerful things (query databases, manage clusters). You must think about:

1. **Principle of least privilege** — Only expose read operations unless write is explicitly needed
2. **Input sanitization** — Never pass user input directly to shell commands or SQL
3. **Audit logging** — Log every tool call with timestamp, arguments, and result summary
4. **Rate limiting** — Prevent runaway tool calls from overwhelming your systems
5. **Secret management** — API keys and credentials via environment variables, never hardcoded

---

## 🔨 Hands-On (50 min)

### Exercise 1: Kubernetes MCP Server (20 min)

**Goal:** Build an MCP server that gives Claude access to Kubernetes cluster information. Read-only operations only (safe for production).

Create `week12/k8s_mcp_server.py`:

```python
"""
Week 12, Exercise 1: Kubernetes MCP Server

Gives Claude read-only access to Kubernetes cluster information.
In production, this calls real kubectl commands.
Here we simulate responses for safety and portability.

Tools:
  - list_pods: List pods in a namespace with status
  - get_pod_logs: Get logs from a specific pod
  - describe_pod: Get detailed pod information
  - list_deployments: List deployments with replica counts
  - get_events: Get recent cluster events

Security:
  - Read-only operations only (no create, delete, scale)
  - Namespace restriction (can only access allowed namespaces)
  - Input validation on all arguments
  - Audit logging of every tool call
"""
import json
import logging
import re
from datetime import datetime
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

# ============================================================
# CONFIGURATION
# ============================================================
ALLOWED_NAMESPACES = ["payments", "auth", "monitoring", "default"]
LOG_FILE = "k8s_mcp_audit.log"

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s | %(levelname)s | %(message)s',
    handlers=[
        logging.FileHandler(LOG_FILE),
        logging.StreamHandler()  # Also print to stderr for debugging
    ]
)
logger = logging.getLogger("k8s-mcp")

server = Server("kubernetes-sre")


# ============================================================
# INPUT VALIDATION
# ============================================================
def validate_namespace(namespace: str) -> str:
    """Validate and sanitize namespace input."""
    ns = namespace.strip().lower()
    if ns not in ALLOWED_NAMESPACES:
        raise ValueError(f"Namespace '{ns}' not allowed. Allowed: {ALLOWED_NAMESPACES}")
    return ns

def validate_pod_name(name: str) -> str:
    """Validate pod name format."""
    if not re.match(r'^[a-z0-9]([a-z0-9\-\.]*[a-z0-9])?$', name):
        raise ValueError(f"Invalid pod name format: '{name}'")
    return name

def validate_lines(lines: int) -> int:
    """Validate log line count."""
    return max(1, min(lines, 500))  # Clamp between 1 and 500


# ============================================================
# SIMULATED KUBERNETES DATA
# ============================================================
CLUSTER_DATA = {
    "payments": {
        "pods": [
            {"name": "payment-gw-abc12", "status": "Running", "ready": "1/1", "restarts": 0, "age": "3d", "node": "worker-01", "cpu": "250m", "memory": "384Mi"},
            {"name": "payment-gw-def34", "status": "CrashLoopBackOff", "ready": "0/1", "restarts": 47, "age": "3d", "node": "worker-02", "cpu": "0m", "memory": "512Mi"},
            {"name": "payment-gw-ghi56", "status": "CrashLoopBackOff", "ready": "0/1", "restarts": 32, "age": "3d", "node": "worker-01", "cpu": "0m", "memory": "510Mi"},
            {"name": "payment-gw-jkl78", "status": "Running", "ready": "1/1", "restarts": 0, "age": "3d", "node": "worker-03", "cpu": "230m", "memory": "360Mi"},
            {"name": "payment-gw-mno90", "status": "Running", "ready": "1/1", "restarts": 0, "age": "3d", "node": "worker-02", "cpu": "280m", "memory": "390Mi"},
            {"name": "redis-primary-0", "status": "Running", "ready": "1/1", "restarts": 0, "age": "14d", "node": "worker-03", "cpu": "180m", "memory": "2.1Gi"},
            {"name": "redis-replica-0", "status": "Running", "ready": "1/1", "restarts": 0, "age": "14d", "node": "worker-01", "cpu": "120m", "memory": "1.9Gi"},
        ],
        "deployments": [
            {"name": "payment-gateway", "ready": "3/5", "up_to_date": 5, "available": 3, "age": "45d", "image": "visa/payment-gw:v2.3.1"},
        ],
        "events": [
            {"time": "14:23", "type": "Warning", "reason": "OOMKilled", "object": "pod/payment-gw-def34", "message": "Container payment-svc exceeded memory limit (512Mi)"},
            {"time": "14:22", "type": "Warning", "reason": "BackOff", "object": "pod/payment-gw-def34", "message": "Back-off restarting failed container"},
            {"time": "14:22", "type": "Warning", "reason": "OOMKilled", "object": "pod/payment-gw-ghi56", "message": "Container payment-svc exceeded memory limit (512Mi)"},
            {"time": "14:20", "type": "Normal", "reason": "Pulled", "object": "pod/payment-gw-def34", "message": "Successfully pulled image visa/payment-gw:v2.3.1"},
            {"time": "14:00", "type": "Normal", "reason": "ScalingReplicaSet", "object": "deployment/payment-gateway", "message": "Scaled up replica set payment-gw-v2.3.1 to 5"},
        ],
        "pod_logs": {
            "payment-gw-def34": [
                "14:23:01 INFO  [Main] Starting payment-gateway v2.3.1",
                "14:23:02 INFO  [Config] Loading application config from ConfigMap",
                "14:23:03 INFO  [DB] Connected to pg-primary:5432",
                "14:23:04 INFO  [Redis] Connected to redis-primary:6379",
                "14:23:05 INFO  [BatchReconciliation] Starting daily reconciliation job",
                "14:23:06 WARN  [BatchReconciliation] Loading 2.1M transactions into memory for reconciliation",
                "14:23:08 WARN  [GC] Full GC triggered — heap usage at 92%",
                "14:23:10 WARN  [GC] Full GC triggered — heap usage at 97%",
                "14:23:11 ERROR [BatchReconciliation] java.lang.OutOfMemoryError: Java heap space",
                "14:23:11 ERROR [BatchReconciliation]   at java.util.ArrayList.grow(ArrayList.java:265)",
                "14:23:11 ERROR [BatchReconciliation]   at com.visa.payment.BatchReconciliation.loadAllTransactions(BatchReconciliation.java:142)",
                "14:23:11 ERROR [BatchReconciliation]   at com.visa.payment.BatchReconciliation.run(BatchReconciliation.java:87)",
                "14:23:11 ERROR [Main] Uncaught exception in thread main — shutting down",
            ],
            "payment-gw-abc12": [
                "14:00:01 INFO  [Main] Starting payment-gateway v2.3.1",
                "14:00:02 INFO  [Config] Loading application config",
                "14:00:03 INFO  [DB] Connected to pg-primary:5432",
                "14:00:04 INFO  [Redis] Connected to redis-primary:6379",
                "14:00:05 INFO  [HealthCheck] All checks passed",
                "14:15:00 INFO  [HealthCheck] All checks passed",
                "14:23:00 INFO  [HealthCheck] All checks passed",
            ]
        },
        "pod_describe": {
            "payment-gw-def34": {
                "name": "payment-gw-def34",
                "namespace": "payments",
                "node": "worker-02",
                "status": "CrashLoopBackOff",
                "ip": "10.244.2.15",
                "containers": [{
                    "name": "payment-svc",
                    "image": "visa/payment-gw:v2.3.1",
                    "state": "Waiting (CrashLoopBackOff)",
                    "last_state": "Terminated (OOMKilled, exit code 137)",
                    "ready": False,
                    "restart_count": 47,
                    "resources": {
                        "requests": {"cpu": "200m", "memory": "256Mi"},
                        "limits": {"cpu": "500m", "memory": "512Mi"}
                    },
                    "liveness_probe": "HTTP GET /health :8080 delay=10s timeout=3s period=10s",
                }],
                "conditions": [
                    {"type": "Ready", "status": "False", "reason": "ContainersNotReady"},
                    {"type": "ContainersReady", "status": "False"},
                ],
                "events": [
                    "Warning OOMKilled: Container exceeded memory limit (512Mi)",
                    "Warning BackOff: Back-off restarting failed container",
                    "Normal  Pulled: Successfully pulled image",
                ]
            }
        }
    }
}


# ============================================================
# TOOL IMPLEMENTATIONS
# ============================================================

def list_pods(namespace: str, status_filter: str = None) -> dict:
    ns = validate_namespace(namespace)
    pods = CLUSTER_DATA.get(ns, {}).get("pods", [])
    
    if status_filter:
        pods = [p for p in pods if p["status"].lower() == status_filter.lower()]
    
    return {
        "namespace": ns,
        "pod_count": len(pods),
        "pods": pods,
        "summary": {
            "running": sum(1 for p in CLUSTER_DATA.get(ns, {}).get("pods", []) if p["status"] == "Running"),
            "not_ready": sum(1 for p in CLUSTER_DATA.get(ns, {}).get("pods", []) if p["status"] != "Running"),
        }
    }

def get_pod_logs(namespace: str, pod_name: str, lines: int = 20, previous: bool = False) -> dict:
    ns = validate_namespace(namespace)
    pod_name = validate_pod_name(pod_name)
    lines = validate_lines(lines)
    
    all_logs = CLUSTER_DATA.get(ns, {}).get("pod_logs", {})
    pod_logs = all_logs.get(pod_name, [])
    
    if not pod_logs:
        return {"error": f"No logs found for pod '{pod_name}' in namespace '{ns}'",
                "available_pods": list(all_logs.keys())}
    
    return {
        "namespace": ns,
        "pod": pod_name,
        "previous_container": previous,
        "lines": pod_logs[-lines:],
        "total_lines": len(pod_logs)
    }

def describe_pod(namespace: str, pod_name: str) -> dict:
    ns = validate_namespace(namespace)
    pod_name = validate_pod_name(pod_name)
    
    describes = CLUSTER_DATA.get(ns, {}).get("pod_describe", {})
    if pod_name in describes:
        return describes[pod_name]
    return {"error": f"Pod '{pod_name}' not found in namespace '{ns}'"}

def list_deployments(namespace: str) -> dict:
    ns = validate_namespace(namespace)
    return {
        "namespace": ns,
        "deployments": CLUSTER_DATA.get(ns, {}).get("deployments", [])
    }

def get_events(namespace: str, event_type: str = None) -> dict:
    ns = validate_namespace(namespace)
    events = CLUSTER_DATA.get(ns, {}).get("events", [])
    
    if event_type:
        events = [e for e in events if e["type"].lower() == event_type.lower()]
    
    return {"namespace": ns, "events": events, "count": len(events)}


# ============================================================
# MCP TOOL REGISTRATION
# ============================================================

@server.list_tools()
async def handle_list_tools() -> list[Tool]:
    return [
        Tool(
            name="list_pods",
            description="List all pods in a Kubernetes namespace with their status, restarts, and resource usage. Use as a first step to see what's running and what's failing.",
            inputSchema={
                "type": "object",
                "properties": {
                    "namespace": {"type": "string", "description": f"Kubernetes namespace. Allowed: {ALLOWED_NAMESPACES}"},
                    "status_filter": {"type": "string", "description": "Filter by status: Running, CrashLoopBackOff, Pending, etc.", "default": None}
                },
                "required": ["namespace"]
            }
        ),
        Tool(
            name="get_pod_logs",
            description="Get recent log lines from a specific pod. Use to see error messages, stack traces, and application behavior. Specify 'previous=true' to get logs from the last crashed container.",
            inputSchema={
                "type": "object",
                "properties": {
                    "namespace": {"type": "string"},
                    "pod_name": {"type": "string", "description": "Exact pod name from list_pods output"},
                    "lines": {"type": "integer", "description": "Number of log lines (1-500, default 20)", "default": 20},
                    "previous": {"type": "boolean", "description": "Get logs from previous (crashed) container", "default": False}
                },
                "required": ["namespace", "pod_name"]
            }
        ),
        Tool(
            name="describe_pod",
            description="Get detailed information about a pod: containers, resources, probes, conditions, and recent events. Use when you need to understand WHY a pod is failing — shows OOMKilled, image errors, probe failures.",
            inputSchema={
                "type": "object",
                "properties": {
                    "namespace": {"type": "string"},
                    "pod_name": {"type": "string", "description": "Exact pod name"}
                },
                "required": ["namespace", "pod_name"]
            }
        ),
        Tool(
            name="list_deployments",
            description="List deployments in a namespace with replica counts and current image versions. Shows ready vs desired replicas.",
            inputSchema={
                "type": "object",
                "properties": {
                    "namespace": {"type": "string"}
                },
                "required": ["namespace"]
            }
        ),
        Tool(
            name="get_events",
            description="Get recent Kubernetes events in a namespace. Shows warnings, errors, scaling events, and scheduling issues. Filter by type: Warning or Normal.",
            inputSchema={
                "type": "object",
                "properties": {
                    "namespace": {"type": "string"},
                    "event_type": {"type": "string", "enum": ["Warning", "Normal"], "description": "Filter by event type"}
                },
                "required": ["namespace"]
            }
        ),
    ]


@server.call_tool()
async def handle_call_tool(name: str, arguments: dict) -> list[TextContent]:
    logger.info(f"TOOL_CALL | {name} | args={json.dumps(arguments)}")
    
    try:
        if name == "list_pods":
            result = list_pods(arguments["namespace"], arguments.get("status_filter"))
        elif name == "get_pod_logs":
            result = get_pod_logs(
                arguments["namespace"], arguments["pod_name"],
                arguments.get("lines", 20), arguments.get("previous", False)
            )
        elif name == "describe_pod":
            result = describe_pod(arguments["namespace"], arguments["pod_name"])
        elif name == "list_deployments":
            result = list_deployments(arguments["namespace"])
        elif name == "get_events":
            result = get_events(arguments["namespace"], arguments.get("event_type"))
        else:
            result = {"error": f"Unknown tool: {name}"}
        
        logger.info(f"TOOL_RESULT | {name} | success")
        return [TextContent(type="text", text=json.dumps(result, indent=2))]
    
    except ValueError as e:
        logger.warning(f"TOOL_VALIDATION_ERROR | {name} | {e}")
        return [TextContent(type="text", text=json.dumps({"error": f"Validation error: {str(e)}"}))]
    
    except Exception as e:
        logger.error(f"TOOL_ERROR | {name} | {e}")
        return [TextContent(type="text", text=json.dumps({"error": f"Internal error: {str(e)}"}))]


# ============================================================
# RUN SERVER
# ============================================================
async def main():
    async with stdio_server() as (read_stream, write_stream):
        logger.info("Kubernetes MCP server starting...")
        await server.run(read_stream, write_stream, server.create_initialization_options())

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

---

### Exercise 2: ClickHouse Log Query MCP Server (20 min)

**Goal:** Build an MCP server that lets Claude query your ClickHouse logs — directly relevant to your Visa Splunk→ClickHouse migration.

Create `week12/clickhouse_mcp_server.py`:

```python
"""
Week 12, Exercise 2: ClickHouse Log Query MCP Server

Gives Claude the ability to search and analyze application logs
stored in ClickHouse. Simulates the Visa migration from Splunk.

Tools:
  - search_logs: Search logs by service, severity, time range, keyword
  - count_errors: Get error counts grouped by service or error type
  - get_slow_queries: Find slow database queries in the logs
  - trace_transaction: Follow a transaction ID across services

Security:
  - Read-only (SELECT only, no INSERT/DELETE/DROP)
  - Query timeout enforcement
  - Result size limits
"""
import json
import logging
import re
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

logging.basicConfig(level=logging.INFO, format='%(asctime)s | %(message)s')
logger = logging.getLogger("clickhouse-mcp")

server = Server("clickhouse-logs")


# ============================================================
# SIMULATED CLICKHOUSE DATA
# ============================================================
# In production, these functions execute actual ClickHouse SQL queries.
# Here we simulate the results.

LOG_DATA = [
    {"timestamp": "2024-10-15 14:23:01.234", "service": "payment-gateway", "level": "ERROR", "message": "java.lang.OutOfMemoryError: Java heap space", "trace_id": "tx-abc123", "host": "payment-gw-def34"},
    {"timestamp": "2024-10-15 14:23:01.567", "service": "payment-gateway", "level": "ERROR", "message": "BatchReconciliation.loadAllTransactions failed", "trace_id": "tx-abc123", "host": "payment-gw-def34"},
    {"timestamp": "2024-10-15 14:22:58.890", "service": "payment-gateway", "level": "WARN", "message": "GC Full pause 2.3s — heap at 97%", "trace_id": "", "host": "payment-gw-def34"},
    {"timestamp": "2024-10-15 14:22:55.123", "service": "payment-gateway", "level": "WARN", "message": "Loading 2.1M transactions into memory for reconciliation", "trace_id": "", "host": "payment-gw-def34"},
    {"timestamp": "2024-10-15 14:23:02.456", "service": "payment-gateway", "level": "ERROR", "message": "Transaction tx-def456 failed: upstream timeout (30s)", "trace_id": "tx-def456", "host": "payment-gw-abc12"},
    {"timestamp": "2024-10-15 14:23:03.789", "service": "payment-gateway", "level": "ERROR", "message": "Circuit breaker OPEN for card-network-gateway (12/10 failures)", "trace_id": "", "host": "payment-gw-abc12"},
    {"timestamp": "2024-10-15 14:22:00.100", "service": "auth-service", "level": "INFO", "message": "Health check passed, latency 12ms", "trace_id": "", "host": "auth-svc-xyz99"},
    {"timestamp": "2024-10-15 14:22:30.200", "service": "auth-service", "level": "WARN", "message": "JWT token expires in 4 hours for service account batch-sa", "trace_id": "", "host": "auth-svc-xyz99"},
    {"timestamp": "2024-10-15 14:23:00.300", "service": "fraud-detection", "level": "WARN", "message": "Model inference latency 890ms (threshold 500ms)", "trace_id": "tx-abc123", "host": "fraud-ml-01"},
    {"timestamp": "2024-10-15 14:23:01.400", "service": "fraud-detection", "level": "ERROR", "message": "Unable to fetch velocity data from Redis — cache miss", "trace_id": "tx-abc123", "host": "fraud-ml-01"},
    {"timestamp": "2024-10-15 14:21:00.500", "service": "payment-gateway", "level": "INFO", "message": "Transaction tx-good01 completed successfully in 145ms", "trace_id": "tx-good01", "host": "payment-gw-abc12"},
    {"timestamp": "2024-10-15 14:21:30.600", "service": "payment-gateway", "level": "INFO", "message": "Transaction tx-good02 completed successfully in 167ms", "trace_id": "tx-good02", "host": "payment-gw-mno90"},
]

SLOW_QUERIES = [
    {"timestamp": "2024-10-15 14:15:00", "query": "SELECT * FROM transactions WHERE created_at > '2024-10-14'", "duration_ms": 12500, "rows_examined": 2100000, "service": "payment-gateway"},
    {"timestamp": "2024-10-15 14:10:00", "query": "SELECT t.*, c.name FROM transactions t JOIN customers c ON t.customer_id = c.id WHERE t.status = 'pending'", "duration_ms": 8700, "rows_examined": 450000, "service": "payment-gateway"},
    {"timestamp": "2024-10-15 13:55:00", "query": "UPDATE fraud_scores SET recalculated=true WHERE checked_at < now() - INTERVAL 24 HOUR", "duration_ms": 5200, "rows_examined": 180000, "service": "fraud-detection"},
]


# ============================================================
# TOOL IMPLEMENTATIONS
# ============================================================

def validate_sql_safe(keyword: str) -> str:
    """Ensure keyword doesn't contain SQL injection patterns."""
    if keyword and re.search(r'(DROP|DELETE|INSERT|UPDATE|ALTER|GRANT|EXEC|;|--)', keyword, re.IGNORECASE):
        raise ValueError(f"Suspicious input rejected: '{keyword}'")
    return keyword

def search_logs(service: str = None, level: str = None, keyword: str = None,
                time_range: str = "1h", limit: int = 20) -> dict:
    """Search logs with filters."""
    if keyword:
        validate_sql_safe(keyword)
    
    results = LOG_DATA.copy()
    
    if service:
        results = [r for r in results if r["service"] == service]
    if level:
        results = [r for r in results if r["level"] == level.upper()]
    if keyword:
        kw = keyword.lower()
        results = [r for r in results if kw in r["message"].lower()]
    
    results = results[:min(limit, 100)]  # Cap at 100
    
    return {
        "query_filters": {"service": service, "level": level, "keyword": keyword, "time_range": time_range},
        "result_count": len(results),
        "logs": results
    }

def count_errors(group_by: str = "service", time_range: str = "1h") -> dict:
    """Count errors grouped by field."""
    errors = [r for r in LOG_DATA if r["level"] == "ERROR"]
    
    counts = {}
    for e in errors:
        if group_by == "service":
            key = e["service"]
        elif group_by == "host":
            key = e["host"]
        elif group_by == "message":
            # Group by first 50 chars of message (error type)
            key = e["message"][:50]
        else:
            key = e.get(group_by, "unknown")
        
        counts[key] = counts.get(key, 0) + 1
    
    sorted_counts = sorted(counts.items(), key=lambda x: x[1], reverse=True)
    
    return {
        "group_by": group_by,
        "time_range": time_range,
        "total_errors": len(errors),
        "groups": [{"key": k, "count": v} for k, v in sorted_counts]
    }

def get_slow_queries_fn(service: str = None, min_duration_ms: int = 1000, limit: int = 10) -> dict:
    """Find slow database queries in logs."""
    results = SLOW_QUERIES.copy()
    
    if service:
        results = [r for r in results if r["service"] == service]
    
    results = [r for r in results if r["duration_ms"] >= min_duration_ms]
    results.sort(key=lambda x: x["duration_ms"], reverse=True)
    
    return {
        "filters": {"service": service, "min_duration_ms": min_duration_ms},
        "count": len(results[:limit]),
        "slow_queries": results[:limit]
    }

def trace_transaction(trace_id: str) -> dict:
    """Follow a transaction across all services."""
    validate_sql_safe(trace_id)
    
    events = [r for r in LOG_DATA if r["trace_id"] == trace_id]
    events.sort(key=lambda x: x["timestamp"])
    
    services_involved = list(set(e["service"] for e in events))
    has_error = any(e["level"] == "ERROR" for e in events)
    
    return {
        "trace_id": trace_id,
        "services": services_involved,
        "event_count": len(events),
        "has_error": has_error,
        "events": events
    }


# ============================================================
# MCP REGISTRATION
# ============================================================

@server.list_tools()
async def handle_list_tools() -> list[Tool]:
    return [
        Tool(
            name="search_logs",
            description="Search application logs in ClickHouse. Filter by service, severity level, keyword, and time range. Use when investigating errors, finding patterns, or searching for specific events.",
            inputSchema={
                "type": "object",
                "properties": {
                    "service": {"type": "string", "description": "Filter by service name (e.g., 'payment-gateway', 'auth-service', 'fraud-detection')"},
                    "level": {"type": "string", "enum": ["ERROR", "WARN", "INFO"], "description": "Filter by log level"},
                    "keyword": {"type": "string", "description": "Search for this text in log messages"},
                    "time_range": {"type": "string", "description": "Time range: '15m', '1h', '6h', '24h'", "default": "1h"},
                    "limit": {"type": "integer", "description": "Max results (1-100)", "default": 20}
                }
            }
        ),
        Tool(
            name="count_errors",
            description="Count error logs grouped by service, host, or error message. Use to understand error distribution and find the highest-error services or hosts.",
            inputSchema={
                "type": "object",
                "properties": {
                    "group_by": {"type": "string", "enum": ["service", "host", "message"], "description": "Group error counts by this field", "default": "service"},
                    "time_range": {"type": "string", "default": "1h"}
                }
            }
        ),
        Tool(
            name="get_slow_queries",
            description="Find slow database queries logged by services. Shows query text, duration, and rows examined. Use when investigating latency issues.",
            inputSchema={
                "type": "object",
                "properties": {
                    "service": {"type": "string", "description": "Filter by service name"},
                    "min_duration_ms": {"type": "integer", "description": "Minimum query duration in ms", "default": 1000},
                    "limit": {"type": "integer", "default": 10}
                }
            }
        ),
        Tool(
            name="trace_transaction",
            description="Follow a specific transaction across all services by trace ID. Shows the full lifecycle of a payment transaction. Use when debugging why a specific transaction failed.",
            inputSchema={
                "type": "object",
                "properties": {
                    "trace_id": {"type": "string", "description": "Transaction trace ID (e.g., 'tx-abc123')"}
                },
                "required": ["trace_id"]
            }
        ),
    ]


@server.call_tool()
async def handle_call_tool(name: str, arguments: dict) -> list[TextContent]:
    logger.info(f"TOOL_CALL | {name} | {json.dumps(arguments)}")
    
    try:
        if name == "search_logs":
            result = search_logs(**arguments)
        elif name == "count_errors":
            result = count_errors(**arguments)
        elif name == "get_slow_queries":
            result = get_slow_queries_fn(**arguments)
        elif name == "trace_transaction":
            result = trace_transaction(**arguments)
        else:
            result = {"error": f"Unknown tool: {name}"}
        
        return [TextContent(type="text", text=json.dumps(result, indent=2))]
    
    except ValueError as e:
        return [TextContent(type="text", text=json.dumps({"error": str(e)}))]
    except Exception as e:
        logger.error(f"ERROR | {name} | {e}")
        return [TextContent(type="text", text=json.dumps({"error": f"Query failed: {str(e)}"}))]


async def main():
    async with stdio_server() as (read, write):
        logger.info("ClickHouse MCP server starting...")
        await server.run(read, write, server.create_initialization_options())

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

---

### Exercise 3: RAG Knowledge Base as MCP Server (10 min)

**Goal:** Expose your Week 8-10 RAG knowledge base via MCP — Claude Desktop can search your runbooks.

Create `week12/rag_mcp_server.py`:

```python
"""
Week 12, Exercise 3: RAG Knowledge Base as MCP Server

Combines your RAG system with MCP:
- Tool: search_runbooks — semantic search over runbook content
- Tool: ask_knowledge_base — full RAG (search + Claude generates answer)
- Resources: individual runbooks available as browsable resources
"""
import json
import chromadb
from sentence_transformers import SentenceTransformer
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent, Resource

server = Server("sre-knowledge-base")
embedder = SentenceTransformer('all-MiniLM-L6-v2')

# Setup ChromaDB with runbooks
chroma = chromadb.Client()
collection = chroma.get_or_create_collection("runbooks")

RUNBOOKS = {
    "pod-crash": {
        "title": "Pod CrashLoopBackOff Recovery",
        "team": "platform",
        "content": "When a pod is in CrashLoopBackOff: check logs with kubectl logs --previous. Check events with kubectl describe pod. OOMKilled: increase memory limits. Image pull error: check registry credentials. Application crash: check stack traces. Liveness probe failure: adjust probe config. Quick restart: kubectl rollout restart deployment. Rollback: kubectl rollout undo."
    },
    "redis-failure": {
        "title": "Redis Cache Recovery",
        "team": "data",
        "content": "Redis failure: redis-cli ping (expect PONG). Check memory: redis-cli info memory. Full: check maxmemory-policy (allkeys-lru). Connections: redis-cli info clients. Cluster: redis-cli cluster info. Emergency: FLUSHALL if acceptable. Monitor cache hit rate recovery to 90%."
    },
    "db-connections": {
        "title": "PostgreSQL Connection Issues",
        "team": "data",
        "content": "DB connections exhausted: SELECT count(*), state FROM pg_stat_activity GROUP BY state. Kill idle: pg_terminate_backend(pid) WHERE state='idle' AND query_start < now()-interval '10 min'. SHOW max_connections. Long-term: deploy pgbouncer. Pool size: max_connections / pods * 0.8."
    },
    "high-latency": {
        "title": "Service High Latency Investigation",
        "team": "platform",
        "content": "High latency: scope the problem (all endpoints or specific routes?). Check resource saturation: kubectl top pods. Check upstream dependencies: database, Redis, external APIs. Check recent deployments: kubectl rollout history. Traffic anomalies: compare QPS to baseline. Rollback if deploy-related."
    },
    "disk-space": {
        "title": "Disk Space Emergency",
        "team": "platform",
        "content": "Disk full: du -sh /* to identify. Quick cleanup: journalctl --vacuum-time=3d, docker system prune, find /tmp -mtime +7 -delete. Kubernetes: crictl rmi --prune. Log rotation: logrotate -f /etc/logrotate.conf. Long-term: PVC auto-expansion, log rotation policies, alert at 80%."
    }
}

# Load runbooks into ChromaDB
for key, rb in RUNBOOKS.items():
    emb = embedder.encode(f"{rb['title']}\n{rb['content']}").tolist()
    try:
        collection.add(
            ids=[key],
            documents=[rb['content']],
            metadatas=[{"title": rb['title'], "team": rb['team']}],
            embeddings=[emb]
        )
    except Exception:
        pass  # Already exists


# ============================================================
# TOOLS
# ============================================================

@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="search_runbooks",
            description="Search SRE runbooks using natural language. Finds the most relevant runbook sections for a given problem. Returns matching runbook content with relevance scores.",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "Natural language description of the problem or question"},
                    "n_results": {"type": "integer", "description": "Number of results (1-5)", "default": 3},
                    "team_filter": {"type": "string", "description": "Filter by team: 'platform', 'data', 'network'"}
                },
                "required": ["query"]
            }
        ),
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    if name == "search_runbooks":
        query = arguments["query"]
        n = min(arguments.get("n_results", 3), 5)
        team = arguments.get("team_filter")
        
        query_emb = embedder.encode(query).tolist()
        where = {"team": team} if team else None
        
        results = collection.query(
            query_embeddings=[query_emb],
            n_results=n,
            where=where,
            include=["documents", "metadatas", "distances"]
        )
        
        formatted = []
        for i in range(len(results['ids'][0])):
            formatted.append({
                "title": results['metadatas'][0][i]['title'],
                "team": results['metadatas'][0][i]['team'],
                "relevance_score": round(1 - results['distances'][0][i] / 2, 3),
                "content": results['documents'][0][i]
            })
        
        return [TextContent(type="text", text=json.dumps({"results": formatted}, indent=2))]
    
    return [TextContent(type="text", text=json.dumps({"error": f"Unknown tool: {name}"}))]


# ============================================================
# RESOURCES — browsable runbooks
# ============================================================

@server.list_resources()
async def list_resources() -> list[Resource]:
    return [
        Resource(
            uri=f"runbook://{key}",
            name=rb["title"],
            description=f"SRE runbook: {rb['title']} ({rb['team']} team)",
            mimeType="text/plain"
        )
        for key, rb in RUNBOOKS.items()
    ]

@server.read_resource()
async def read_resource(uri: str) -> str:
    key = uri.replace("runbook://", "")
    if key in RUNBOOKS:
        rb = RUNBOOKS[key]
        return f"# {rb['title']}\nTeam: {rb['team']}\n\n{rb['content']}"
    raise ValueError(f"Runbook not found: {key}")


# ============================================================
# RUN
# ============================================================
async def main():
    async with stdio_server() as (read, write):
        await server.run(read, write, server.create_initialization_options())

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

**Claude Desktop config with all 3 servers:**

```json
{
  "mcpServers": {
    "kubernetes": {
      "command": "python",
      "args": ["/path/to/week12/k8s_mcp_server.py"]
    },
    "clickhouse-logs": {
      "command": "python",
      "args": ["/path/to/week12/clickhouse_mcp_server.py"]
    },
    "sre-knowledge-base": {
      "command": "python",
      "args": ["/path/to/week12/rag_mcp_server.py"]
    }
  }
}
```

Now Claude Desktop has access to Kubernetes, ClickHouse logs, AND your runbook knowledge base simultaneously!

---

## 📝 Drills (20 min)

### Drill 1: Add Write Operations (with confirmation)

Add a `scale_deployment` tool to the Kubernetes server, but with a safety pattern:

```python
Tool(
    name="scale_deployment",
    description="Scale a deployment to a new replica count. REQUIRES CONFIRMATION: returns a preview of what will change, and a confirmation_token. Call again with the token to execute.",
    inputSchema={...}
)
```

Implement a two-step pattern: first call returns a preview, second call (with token) executes. This prevents accidental scaling.

### Drill 2: Connect to Real ClickHouse

If you have ClickHouse access at work, replace the simulated data with real queries:

```python
from clickhouse_driver import Client as CHClient

ch = CHClient(host='localhost', port=9000)

def search_logs(service, level, keyword, time_range, limit):
    query = f"""
        SELECT timestamp, service, level, message, trace_id
        FROM application_logs
        WHERE timestamp > now() - INTERVAL {time_range}
        {'AND service = %(service)s' if service else ''}
        {'AND level = %(level)s' if level else ''}
        {'AND message ILIKE %(keyword)s' if keyword else ''}
        ORDER BY timestamp DESC
        LIMIT %(limit)s
    """
    params = {"service": service, "level": level, "keyword": f"%{keyword}%", "limit": limit}
    return ch.execute(query, params)
```

### Drill 3: MCP Server Testing Framework

Create `week12/test_mcp_server.py`:

```python
"""Test your MCP server tools without Claude Desktop."""
import asyncio
from k8s_mcp_server import list_pods, get_pod_logs, describe_pod

async def test_all_tools():
    # Test list_pods
    result = list_pods("payments")
    assert result["pod_count"] > 0, "Should find pods"
    assert result["summary"]["not_ready"] > 0, "Should have failing pods"
    
    # Test get_pod_logs
    result = get_pod_logs("payments", "payment-gw-def34")
    assert "OutOfMemoryError" in str(result), "Should find OOM errors"
    
    # Test namespace validation
    try:
        list_pods("hacker-namespace")
        assert False, "Should have rejected invalid namespace"
    except ValueError:
        pass  # Expected
    
    print("✅ All tests passed!")

asyncio.run(test_all_tools())
```

### Drill 4: Multi-Server Investigation

With all 3 servers connected to Claude Desktop, ask:

> "The payment gateway seems degraded. Use all available tools to investigate: check Kubernetes pod status, search the logs, and find any relevant runbooks."

Document Claude's investigation path: which servers it calls, in what order, and how it synthesizes information from all three sources.

---

## ✅ Week 12 Checklist

Before moving to Week 13:

- [ ] Have a Kubernetes MCP server with 5 read-only tools + input validation + audit logging
- [ ] Have a ClickHouse MCP server with 4 log query tools + SQL injection protection
- [ ] Have your RAG knowledge base exposed via MCP (tool + resources)
- [ ] All servers have proper error handling (try/catch, structured error returns)
- [ ] Understand MCP security: namespace restriction, input validation, audit logs
- [ ] Can configure Claude Desktop to use multiple MCP servers simultaneously

---

## 🧠 Key Concepts from This Week

### Production MCP Server Checklist

```
✅ Input validation on ALL arguments
✅ Error handling (never crash — return error JSON)
✅ Audit logging (every tool call with timestamp and arguments)
✅ Security (read-only by default, namespace restrictions, SQL injection prevention)
✅ Configuration via environment variables
✅ Tool descriptions are detailed and accurate
✅ Enum types for constrained inputs
✅ Result size limits (no returning 1M rows)
```

### MCP Server Portfolio

```
By end of this week, you have 3 MCP servers:
  1. kubernetes-sre:       5 tools for cluster investigation
  2. clickhouse-logs:      4 tools for log analysis
  3. sre-knowledge-base:   1 tool + 5 resources for runbook access

Combined: Claude has access to cluster state, application logs, AND runbooks
          in a single conversation. This is the foundation of the
          autonomous incident triage agent (Week 20 capstone).
```

---

*Next week: Advanced MCP — composition, multi-server workflows, MCP + RAG integration, building the `sre-mcp-server` portfolio project.*
