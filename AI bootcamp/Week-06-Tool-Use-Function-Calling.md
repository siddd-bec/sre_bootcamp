# Week 6: Tool Use (Function Calling)

## 🎯 Learning Objectives

By the end of this week, you will:
- Understand tool use — what it is, why it matters, how it works under the hood
- Define tools with JSON schemas that Claude can call
- Implement the agentic tool-use loop (call → execute → return → repeat)
- Build a simulated SRE investigation system where Claude autonomously gathers data
- Handle tool errors gracefully and understand tool_choice modes
- See why tool use is the foundation of everything in Weeks 14-16 (AI Agents)

---

## 📖 Theory (20 min)

### What is Tool Use?

Tool use (also called "function calling") lets Claude **call functions YOU define**. Instead of just generating text, Claude can decide: "I need to check the database" → call your `check_database()` function → get real results → use those results in its response.

**Without tool use:**
```
You: "Is the payment gateway healthy?"
Claude: "I don't have access to your systems, but here are some things you could check..."
```

**With tool use:**
```
You: "Is the payment gateway healthy?"
Claude: [decides to call check_service_health("payment-gateway")]
        [gets back: {"status": "degraded", "error_rate": 5.2%, "pods": "3/5 running"}]
        "The payment gateway is degraded. Error rate is 5.2% and only 3 of 5 pods are running.
         I recommend checking the CrashLoopBackOff pods immediately."
```

Claude made a **decision** about what information it needed, called your function to get it, and incorporated real data into its response. This is a massive leap from pure text generation.

### How Tool Use Works — The Complete Flow

```
Step 1: YOU define tools (name, description, parameters)
              ↓
Step 2: YOU send a message + tools to Claude
              ↓
Step 3: CLAUDE decides which tool(s) to call (or none)
              ↓
Step 4: Claude returns stop_reason="tool_use" + tool call details
              ↓
Step 5: YOUR CODE executes the tool function with Claude's chosen parameters
              ↓
Step 6: YOU send the tool result back to Claude
              ↓
Step 7: Claude uses the result to continue (might call MORE tools or give final answer)
              ↓
Step 8: Repeat steps 3-7 until Claude returns stop_reason="end_turn"
```

This loop (steps 3-7) is called the **agentic loop** — it's the exact same pattern used by AI Agents in Weeks 14-16, just with more tools and more sophisticated reasoning.

### Tool Definition — The JSON Schema

Each tool needs three things:

```python
{
    "name": "check_pod_status",              # What the tool is called
    "description": "Check Kubernetes pod...", # CRITICAL: Claude uses this to decide 
                                              # WHEN to call the tool. Be detailed!
    "input_schema": {                         # What parameters the tool accepts
        "type": "object",
        "properties": {
            "namespace": {
                "type": "string",
                "description": "Kubernetes namespace to check"
            },
            "label_selector": {
                "type": "string",
                "description": "Label filter, e.g. 'app=payment-gateway'"
            }
        },
        "required": ["namespace"]
    }
}
```

**The description is the most important part.** Claude reads it to decide whether to call the tool. A bad description = Claude won't know when to use it.

### tool_choice — Controlling When Claude Uses Tools

| Mode | Behavior | Use When |
|------|----------|---------|
| `{"type": "auto"}` | Claude decides whether to use tools (default) | Most cases |
| `{"type": "any"}` | Claude MUST use at least one tool | Force tool use for structured extraction |
| `{"type": "tool", "name": "X"}` | Claude MUST use this specific tool | Force a particular tool call |

### Why This Matters for SRE

Tool use turns Claude from a "smart text generator" into an **active investigator**. Instead of telling you what to check, Claude actually checks it:

- Query ClickHouse for error logs
- Check Kubernetes pod status
- Look up recent deployments
- Read metrics from Prometheus
- Search runbooks in your wiki
- Create Jira tickets
- Send Slack notifications

In production, your tools connect to real systems. In this bootcamp, we'll use simulated tools first (so you don't need live infrastructure), then connect real tools in later weeks.

---

## 🔨 Hands-On (50 min)

### Exercise 1: Your First Tool — Step by Step (15 min)

**Goal:** Define a single tool, let Claude call it, and handle the result. Understand every step of the flow.

Create `week6/first_tool.py`:

```python
"""
Week 6, Exercise 1: Your First Tool Call

We'll go through the tool use flow step by step, with verbose output
so you can see exactly what happens at each stage.
"""
import anthropic
import json

client = anthropic.Anthropic()

# ============================================================
# STEP 1: Define a tool
# ============================================================
# This tells Claude: "You have access to a function called get_service_status
# that can check if a service is healthy."

tools = [
    {
        "name": "get_service_status",
        "description": (
            "Check the current health status of a production service. "
            "Returns status (healthy/degraded/down), error rate, latency, "
            "and pod count. Use this whenever someone asks about service health "
            "or when investigating an issue with a specific service."
        ),
        "input_schema": {
            "type": "object",
            "properties": {
                "service_name": {
                    "type": "string",
                    "description": "Name of the service to check (e.g., 'payment-gateway', 'auth-service', 'fraud-detection')"
                }
            },
            "required": ["service_name"]
        }
    }
]

# ============================================================
# STEP 2: Implement the tool (what actually runs when called)
# ============================================================
# In production, this would call Kubernetes API, Prometheus, etc.
# For learning, we simulate the responses.

def get_service_status(service_name: str) -> dict:
    """Simulated service health check."""
    services = {
        "payment-gateway": {
            "status": "degraded",
            "error_rate_percent": 4.7,
            "p99_latency_ms": 1850,
            "pods_running": 3,
            "pods_desired": 5,
            "last_deployment": "v2.3.1 deployed 15 min ago",
            "alerts_active": ["ErrorRateHigh", "PodCrashLoopBackOff"]
        },
        "auth-service": {
            "status": "healthy",
            "error_rate_percent": 0.01,
            "p99_latency_ms": 42,
            "pods_running": 4,
            "pods_desired": 4,
            "last_deployment": "v1.8.0 deployed 3 days ago",
            "alerts_active": []
        },
        "fraud-detection": {
            "status": "healthy",
            "error_rate_percent": 0.05,
            "p99_latency_ms": 230,
            "pods_running": 6,
            "pods_desired": 6,
            "last_deployment": "v3.1.2 deployed 1 day ago",
            "alerts_active": []
        }
    }
    
    if service_name in services:
        return services[service_name]
    else:
        return {"error": f"Service '{service_name}' not found. Available: {list(services.keys())}"}


# ============================================================
# STEP 3: Send message + tools to Claude
# ============================================================
print("=" * 70)
print("STEP 3: Sending question to Claude (with tools available)")
print("=" * 70)

messages = [
    {"role": "user", "content": "Is the payment gateway healthy? I've been getting customer complaints."}
]

response = client.messages.create(
    model="claude-sonnet-4-5-20250514",
    max_tokens=1024,
    temperature=0,
    system="You are an SRE assistant with access to production monitoring tools. Use them to investigate issues.",
    tools=tools,
    messages=messages
)

print(f"\n  Stop reason: {response.stop_reason}")
print(f"  Content blocks: {len(response.content)}")

for block in response.content:
    print(f"  Block type: {block.type}")
    if block.type == "text":
        print(f"  Text: {block.text}")
    elif block.type == "tool_use":
        print(f"  Tool call: {block.name}({json.dumps(block.input)})")
        print(f"  Tool use ID: {block.id}")

# ============================================================
# STEP 4: Claude wants to use a tool — execute it
# ============================================================
print(f"\n{'=' * 70}")
print("STEP 4: Claude wants to call a tool — let's execute it")
print("=" * 70)

# Find the tool_use block
tool_block = next(b for b in response.content if b.type == "tool_use")

print(f"\n  Claude decided to call: {tool_block.name}")
print(f"  With parameters: {json.dumps(tool_block.input)}")

# Execute the actual function
tool_result = get_service_status(**tool_block.input)
print(f"  Result: {json.dumps(tool_result, indent=4)}")

# ============================================================
# STEP 5: Send the tool result back to Claude
# ============================================================
print(f"\n{'=' * 70}")
print("STEP 5: Send tool result back to Claude")
print("=" * 70)

# Add Claude's response (with tool_use) to message history
messages.append({"role": "assistant", "content": response.content})

# Add the tool result
messages.append({
    "role": "user",
    "content": [
        {
            "type": "tool_result",
            "tool_use_id": tool_block.id,  # Must match the tool_use block's ID
            "content": json.dumps(tool_result)
        }
    ]
})

# ============================================================
# STEP 6: Claude generates final response using the tool result
# ============================================================
print(f"\n{'=' * 70}")
print("STEP 6: Claude's final answer (using real data)")
print("=" * 70)

final_response = client.messages.create(
    model="claude-sonnet-4-5-20250514",
    max_tokens=1024,
    temperature=0,
    system="You are an SRE assistant with access to production monitoring tools. Use them to investigate issues.",
    tools=tools,
    messages=messages
)

print(f"\n  Stop reason: {final_response.stop_reason}")
for block in final_response.content:
    if block.type == "text":
        print(f"\n{block.text}")

# ============================================================
# KEY INSIGHT
# ============================================================
print(f"\n{'=' * 70}")
print("WHAT JUST HAPPENED")
print("=" * 70)
print("""
1. We told Claude: "Here's a tool you can use to check services"
2. User asked: "Is payment gateway healthy?"
3. Claude DECIDED (on its own) to call get_service_status("payment-gateway")
4. We executed the function and gave Claude the real data
5. Claude used the real data to give a specific, accurate answer

Claude's answer included real numbers (4.7% error rate, 3/5 pods)
because it got them from YOUR function — not from its training data.

This is the fundamental pattern for AI Agents:
  REASON → DECIDE which tool → ACT → OBSERVE result → REASON again
""")
```

---

### Exercise 2: Multi-Tool Investigation (25 min)

**Goal:** Give Claude multiple tools and let it autonomously investigate an issue, calling multiple tools in sequence.

Create `week6/multi_tool_investigation.py`:

```python
"""
Week 6, Exercise 2: Multi-Tool SRE Investigation

Claude gets 5 tools and autonomously decides:
- Which tools to call
- In what order
- What parameters to use
- When it has enough information to answer

This is an AI Agent in its simplest form!
"""
import anthropic
import json

client = anthropic.Anthropic()

# ============================================================
# DEFINE 5 SRE TOOLS
# ============================================================
tools = [
    {
        "name": "check_service_health",
        "description": "Check current health status of a service including error rate, latency, and pod status. Use as a first step when investigating any service issue.",
        "input_schema": {
            "type": "object",
            "properties": {
                "service_name": {
                    "type": "string",
                    "description": "Service to check: 'payment-gateway', 'auth-service', 'fraud-detection', 'notification-service'"
                }
            },
            "required": ["service_name"]
        }
    },
    {
        "name": "get_pod_logs",
        "description": "Retrieve recent logs from a service's pods. Use when you need to see error messages, stack traces, or application-level details. Returns the last 20 log lines.",
        "input_schema": {
            "type": "object",
            "properties": {
                "service_name": {"type": "string", "description": "Service to get logs from"},
                "severity_filter": {
                    "type": "string",
                    "enum": ["ALL", "ERROR", "WARN"],
                    "description": "Filter logs by severity. Default: ERROR"
                }
            },
            "required": ["service_name"]
        }
    },
    {
        "name": "get_recent_deployments",
        "description": "Get deployment history for a service in the last 24 hours. Use when investigating if a recent deployment caused an issue.",
        "input_schema": {
            "type": "object",
            "properties": {
                "service_name": {"type": "string"},
                "hours_back": {
                    "type": "integer",
                    "description": "How many hours of history to retrieve. Default: 24"
                }
            },
            "required": ["service_name"]
        }
    },
    {
        "name": "check_database_health",
        "description": "Check PostgreSQL database health including connections, CPU, replication lag, and slow queries. Use when service issues might be database-related.",
        "input_schema": {
            "type": "object",
            "properties": {
                "database_name": {
                    "type": "string",
                    "description": "Database to check: 'pg-primary', 'pg-replica-01', 'pg-replica-02'"
                }
            },
            "required": ["database_name"]
        }
    },
    {
        "name": "check_redis_health",
        "description": "Check Redis cluster health including memory usage, connection count, hit rate, and eviction rate. Use when cache-related issues are suspected.",
        "input_schema": {
            "type": "object",
            "properties": {
                "cluster_name": {
                    "type": "string",
                    "description": "Redis cluster: 'redis-primary', 'redis-sessions', 'redis-ratelimit'"
                }
            },
            "required": ["cluster_name"]
        }
    }
]

# ============================================================
# IMPLEMENT ALL 5 TOOLS (simulated)
# ============================================================
def check_service_health(service_name: str) -> dict:
    data = {
        "payment-gateway": {
            "status": "degraded", "error_rate": 5.2, "p99_latency_ms": 2100,
            "pods": "3/5 running", "restarts_last_hour": 47,
            "alerts": ["ErrorRateHigh", "PodCrashLoopBackOff"]
        },
        "auth-service": {
            "status": "healthy", "error_rate": 0.02, "p99_latency_ms": 45,
            "pods": "4/4 running", "restarts_last_hour": 0, "alerts": []
        },
        "fraud-detection": {
            "status": "healthy", "error_rate": 0.1, "p99_latency_ms": 180,
            "pods": "6/6 running", "restarts_last_hour": 0, "alerts": []
        }
    }
    return data.get(service_name, {"error": f"Unknown service: {service_name}"})

def get_pod_logs(service_name: str, severity_filter: str = "ERROR") -> dict:
    logs = {
        "payment-gateway": {
            "ERROR": [
                "14:23:01 ERROR [PaymentProcessor] java.lang.OutOfMemoryError: Java heap space",
                "14:23:01 ERROR [PaymentProcessor]   at com.visa.payment.BatchReconciliation.loadAllTransactions(BatchReconciliation.java:142)",
                "14:23:01 ERROR [PaymentProcessor]   at com.visa.payment.BatchReconciliation.run(BatchReconciliation.java:87)",
                "14:22:58 ERROR [PaymentProcessor] java.lang.OutOfMemoryError: Java heap space",
                "14:22:55 ERROR [PaymentProcessor] java.lang.OutOfMemoryError: Java heap space",
                "14:22:50 ERROR [HealthCheck] Liveness probe failed: OOM detected, marking unhealthy",
            ],
            "WARN": [
                "14:23:00 WARN [ConnectionPool] Active connections: 48/50 (96% utilization)",
                "14:22:45 WARN [GC] Full GC triggered, heap usage at 98%",
                "14:22:30 WARN [BatchReconciliation] Loading 2.1M transactions into memory for daily reconciliation",
            ]
        },
        "auth-service": {
            "ERROR": [],
            "WARN": ["14:20:00 WARN [TokenRefresh] Service account token expires in 4 hours"]
        }
    }
    
    service_logs = logs.get(service_name, {"ERROR": [], "WARN": []})
    if severity_filter == "ALL":
        all_logs = service_logs.get("ERROR", []) + service_logs.get("WARN", [])
        return {"service": service_name, "log_lines": all_logs, "count": len(all_logs)}
    else:
        filtered = service_logs.get(severity_filter, [])
        return {"service": service_name, "severity": severity_filter, "log_lines": filtered, "count": len(filtered)}

def get_recent_deployments(service_name: str, hours_back: int = 24) -> dict:
    deployments = {
        "payment-gateway": [
            {"version": "v2.3.1", "time": "14:00 today", "status": "completed", "author": "ci-bot",
             "changes": "Added BatchReconciliation job for daily transaction reconciliation"},
            {"version": "v2.3.0", "time": "yesterday 10:00", "status": "completed", "author": "ci-bot",
             "changes": "Bug fix for card network timeout handling"},
        ],
        "auth-service": [
            {"version": "v1.8.0", "time": "3 days ago", "status": "completed", "author": "ci-bot",
             "changes": "Updated JWT library to v4.2"}
        ]
    }
    return {
        "service": service_name,
        "deployments": deployments.get(service_name, []),
        "count": len(deployments.get(service_name, []))
    }

def check_database_health(database_name: str) -> dict:
    data = {
        "pg-primary": {
            "status": "healthy", "cpu_percent": 42,
            "connections_active": 85, "connections_max": 200,
            "replication_lag_seconds": 0.3,
            "slow_queries_last_hour": 2,
            "disk_usage_percent": 67
        }
    }
    return data.get(database_name, {"error": f"Unknown database: {database_name}"})

def check_redis_health(cluster_name: str) -> dict:
    data = {
        "redis-primary": {
            "status": "healthy", "memory_used_percent": 61,
            "hit_rate": 0.94, "connections": 120,
            "evictions_per_sec": 5, "cluster_state": "ok"
        },
        "redis-ratelimit": {
            "status": "healthy", "memory_used_percent": 45,
            "hit_rate": 0.98, "connections": 80,
            "evictions_per_sec": 0, "cluster_state": "ok"
        }
    }
    return data.get(cluster_name, {"error": f"Unknown cluster: {cluster_name}"})

# Map tool names to functions
TOOL_MAP = {
    "check_service_health": lambda args: check_service_health(**args),
    "get_pod_logs": lambda args: get_pod_logs(**args),
    "get_recent_deployments": lambda args: get_recent_deployments(**args),
    "check_database_health": lambda args: check_database_health(**args),
    "check_redis_health": lambda args: check_redis_health(**args),
}

# ============================================================
# THE AGENTIC LOOP — Let Claude investigate autonomously
# ============================================================
def investigate(question: str, max_turns: int = 10) -> str:
    """
    Let Claude investigate a question by autonomously calling tools.
    
    This is the agentic loop — the foundation of AI Agents.
    Claude keeps calling tools until it has enough info to answer.
    """
    messages = [{"role": "user", "content": question}]
    
    system = """You are a senior SRE investigating a production issue.
You have access to monitoring tools. Use them to gather data before answering.
Be thorough — check multiple systems if needed.
Don't guess — use the tools to get real data.
After gathering enough data, provide a clear analysis with:
1. What's happening (based on data you collected)
2. Root cause
3. Immediate action items with specific commands"""
    
    print(f"\n{'🔍' * 35}")
    print(f"INVESTIGATION: {question}")
    print(f"{'🔍' * 35}\n")
    
    for turn in range(max_turns):
        response = client.messages.create(
            model="claude-sonnet-4-5-20250514",
            max_tokens=2000,
            temperature=0,
            system=system,
            tools=tools,
            messages=messages
        )
        
        # Check if Claude wants to use tools
        if response.stop_reason == "tool_use":
            # Process ALL tool calls in this response
            # (Claude can request multiple tools in one turn)
            tool_results = []
            
            for block in response.content:
                if block.type == "text" and block.text.strip():
                    print(f"💭 Claude thinks: {block.text.strip()[:100]}")
                
                if block.type == "tool_use":
                    tool_name = block.name
                    tool_input = block.input
                    
                    print(f"🔧 [{turn+1}] Calling: {tool_name}({json.dumps(tool_input)})")
                    
                    # Execute the tool
                    if tool_name in TOOL_MAP:
                        result = TOOL_MAP[tool_name](tool_input)
                    else:
                        result = {"error": f"Unknown tool: {tool_name}"}
                    
                    # Show a summary of results
                    result_str = json.dumps(result)
                    if len(result_str) > 120:
                        print(f"   📊 Result: {result_str[:120]}...")
                    else:
                        print(f"   📊 Result: {result_str}")
                    
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": json.dumps(result)
                    })
            
            # Add Claude's response and tool results to history
            messages.append({"role": "assistant", "content": response.content})
            messages.append({"role": "user", "content": tool_results})
            print()  # Blank line between turns
        
        else:
            # Claude is done — print final analysis
            print(f"\n{'=' * 70}")
            print(f"📋 FINAL ANALYSIS (after {turn+1} turns)")
            print(f"{'=' * 70}\n")
            
            for block in response.content:
                if block.type == "text":
                    print(block.text)
            
            return response.content[0].text if response.content else ""
    
    print(f"\n⚠️  Max turns ({max_turns}) reached without final answer")
    return ""


# ============================================================
# RUN INVESTIGATIONS
# ============================================================

# Investigation 1: Service degradation
result = investigate(
    "The payment gateway seems slow and we're getting customer complaints about failed payments. "
    "Can you investigate what's going on?"
)

print(f"\n\n{'#' * 70}")
print()

# Investigation 2: Different question — Claude picks different tools
result = investigate(
    "We're about to deploy a new version of payment-gateway. "
    "Can you check if all upstream dependencies are healthy before we proceed?"
)
```

---

### Exercise 3: Tool Error Handling & Forced Tool Use (10 min)

**Goal:** Handle cases where tools fail, and learn to force Claude to use specific tools.

Create `week6/tool_errors_and_forcing.py`:

```python
"""
Week 6, Exercise 3: Handling tool errors + forcing tool use

In production, tools can fail:
- Network timeout to Kubernetes API
- Database query timeout
- Permission denied
- Invalid parameters

Claude needs to handle these gracefully.
"""
import anthropic
import json

client = anthropic.Anthropic()

tools = [
    {
        "name": "query_metrics",
        "description": "Query Prometheus metrics for a service. Can fail if Prometheus is unreachable.",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "PromQL query string"},
                "duration": {"type": "string", "description": "Time range, e.g. '1h', '30m'"}
            },
            "required": ["query"]
        }
    },
    {
        "name": "run_kubectl",
        "description": "Run a kubectl command against the production cluster. May fail due to permissions.",
        "input_schema": {
            "type": "object",
            "properties": {
                "command": {"type": "string", "description": "kubectl command WITHOUT the 'kubectl' prefix, e.g. 'get pods -n payments'"}
            },
            "required": ["command"]
        }
    }
]

def query_metrics(query: str, duration: str = "1h") -> dict:
    """Simulated — sometimes fails!"""
    if "error_rate" in query:
        return {"query": query, "result": [{"value": 4.7, "timestamp": "14:23:00"}]}
    else:
        # Simulate a failure
        return {"error": "PromQL query timeout after 30s. Prometheus may be overloaded."}

def run_kubectl(command: str) -> dict:
    """Simulated — sometimes fails!"""
    if "get pods" in command:
        return {"output": "NAME                          READY   STATUS             RESTARTS\npayment-gw-abc12   1/1     Running            0\npayment-gw-def34   0/1     CrashLoopBackOff   47\npayment-gw-ghi56   0/1     CrashLoopBackOff   32"}
    elif "logs" in command:
        return {"output": "java.lang.OutOfMemoryError: Java heap space"}
    elif "describe" in command:
        return {"error": "Error: forbidden: User 'sre-bot' cannot describe pods in namespace 'payments'. Contact cluster admin."}
    else:
        return {"output": f"Command executed: kubectl {command}"}

TOOL_MAP = {
    "query_metrics": lambda args: query_metrics(**args),
    "run_kubectl": lambda args: run_kubectl(**args),
}

# ============================================================
# PART A: Error handling in the agentic loop
# ============================================================
print("=" * 70)
print("PART A: Tool errors — Claude handles them gracefully")
print("=" * 70)

messages = [{"role": "user", "content": "Check the payment-gateway error rate and describe the unhealthy pods."}]

for turn in range(5):
    response = client.messages.create(
        model="claude-sonnet-4-5-20250514",
        max_tokens=1500,
        temperature=0,
        system="You are an SRE with kubectl and Prometheus access. If a tool returns an error, acknowledge it and try an alternative approach.",
        tools=tools,
        messages=messages
    )
    
    if response.stop_reason == "tool_use":
        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                print(f"🔧 Calling: {block.name}({json.dumps(block.input)})")
                result = TOOL_MAP.get(block.name, lambda _: {"error": "Unknown tool"})(block.input)
                
                # Check if the tool returned an error
                if "error" in result:
                    print(f"   ❌ Tool error: {result['error']}")
                else:
                    result_str = json.dumps(result)
                    print(f"   ✅ Result: {result_str[:100]}")
                
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": json.dumps(result),
                    # Mark as error so Claude knows the tool failed
                    "is_error": "error" in result
                })
        
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})
    else:
        print(f"\n📋 Claude's response:\n{response.content[0].text}")
        break

# ============================================================
# PART B: Forcing tool use with tool_choice
# ============================================================
print(f"\n\n{'=' * 70}")
print("PART B: Forcing Claude to use a specific tool")
print("=" * 70)

# Force Claude to query metrics (even if it might not want to)
forced_response = client.messages.create(
    model="claude-sonnet-4-5-20250514",
    max_tokens=500,
    temperature=0,
    system="You are an SRE assistant.",
    tools=tools,
    tool_choice={"type": "tool", "name": "query_metrics"},  # FORCE this tool
    messages=[{"role": "user", "content": "How is the payment gateway doing?"}]
)

print("\nWith tool_choice forcing query_metrics:")
for block in forced_response.content:
    if block.type == "tool_use":
        print(f"  🔧 Forced call: {block.name}({json.dumps(block.input)})")

# Force Claude to use ANY tool
any_response = client.messages.create(
    model="claude-sonnet-4-5-20250514",
    max_tokens=500,
    temperature=0,
    system="You are an SRE assistant.",
    tools=tools,
    tool_choice={"type": "any"},  # Must use at least one tool
    messages=[{"role": "user", "content": "Tell me a joke about Kubernetes."}]
)

print("\nWith tool_choice='any' (even for a joke request):")
for block in any_response.content:
    if block.type == "tool_use":
        print(f"  🔧 Forced call: {block.name}({json.dumps(block.input)})")

print("""
💡 tool_choice modes:
  "auto" (default) — Claude decides if/which tools to use
  "any"            — Claude MUST use at least one tool  
  {"type":"tool", "name":"X"} — Claude MUST use this specific tool

Use forced tool_choice when:
  - You want structured data extraction (force a classification tool)
  - You need Claude to always check something before answering
  - Testing/debugging your tools
""")
```

---

## 📝 Drills (20 min)

### Drill 1: Add a 6th Tool — ClickHouse Log Query

Add a `query_clickhouse_logs` tool to Exercise 2 that lets Claude search your log database:

```python
{
    "name": "query_clickhouse_logs",
    "description": "Search application logs in ClickHouse. Supports filtering by service, severity, time range, and keyword search.",
    "input_schema": {
        "type": "object",
        "properties": {
            "service": {"type": "string"},
            "severity": {"type": "string", "enum": ["ERROR", "WARN", "INFO", "ALL"]},
            "time_range": {"type": "string", "description": "e.g., 'last 1h', 'last 30m'"},
            "keyword": {"type": "string", "description": "Search for this text in log messages"}
        },
        "required": ["service"]
    }
}
```

Implement the simulated function and test that Claude uses it when asked about recent log patterns.

### Drill 2: Parallel Tool Calls

Claude can request **multiple tools in a single turn** (they appear as multiple `tool_use` blocks in `response.content`). Verify this by asking:

"Check the health of ALL three services: payment-gateway, auth-service, and fraud-detection."

Does Claude call `check_service_health` three times in one turn? Print all tool calls from a single response.

### Drill 3: Tool Use Cost Analysis

Modify the `investigate()` function to track:
- Number of tool calls made
- Number of API turns (each turn = one API call = cost)
- Total input and output tokens
- Estimated cost of the full investigation

Print a cost summary at the end. Compare the cost of a 3-tool investigation vs a 1-tool investigation.

### Drill 4: Build a Tool Registry

Create a reusable `ToolRegistry` class:

```python
class ToolRegistry:
    """Register tools and their handlers in one place."""
    
    def __init__(self):
        self.tools = []        # Tool definitions for the API
        self.handlers = {}     # Tool name → function mapping
    
    def register(self, name, description, parameters, handler):
        """Register a new tool."""
        pass
    
    def get_definitions(self) -> list:
        """Return tool definitions for the API call."""
        pass
    
    def execute(self, tool_name: str, arguments: dict) -> dict:
        """Execute a tool by name."""
        pass
```

This pattern is exactly what MCP servers use (Week 12) and what agent frameworks use (Week 14).

---

## ✅ Week 6 Checklist

Before moving to Week 7:

- [ ] Can define tools with JSON schemas
- [ ] Can implement the agentic loop (call → execute → return → repeat)
- [ ] Understand how Claude decides which tools to call (based on descriptions)
- [ ] Can handle tool errors gracefully (is_error flag)
- [ ] Know the three tool_choice modes (auto, any, forced)
- [ ] Can build a multi-tool investigation that Claude runs autonomously
- [ ] Understand that this pattern IS an AI Agent (Weeks 14-16 scale this up)

---

## 🧠 Key Concepts from This Week

### The Agentic Loop (memorize this!)

```
while True:
    response = call_claude(messages, tools)
    
    if response.stop_reason == "tool_use":
        for tool_call in response.tool_calls:
            result = execute_tool(tool_call)
            send_result_back(result)
    
    elif response.stop_reason == "end_turn":
        print(response.text)
        break
```

This exact pattern powers:
- Week 6: Simple tool use (you are here)
- Week 9: MCP servers (tools exposed via protocol)
- Week 14: ReAct agents (tools + reasoning)
- Week 15: Production agents (tools + memory + planning)
- Week 16: Multi-agent systems (agents calling other agents)
- Week 20: Capstone incident triage agent

### Tool Description Quality = Agent Quality

```
❌ Bad description:  "Check stuff"
   → Claude won't know when to use this tool

✅ Good description: "Check the current health status of a production 
   service including error rate, latency, and pod status. Use as a 
   first step when investigating any service issue."
   → Claude knows exactly when and why to call it
```

### Tool Design Principles

```
1. One tool = one action (don't combine unrelated operations)
2. Clear descriptions (Claude reads them to decide when to call)
3. Specific parameter types (use enums where possible)
4. Return structured data (JSON, not free text)
5. Include error info in returns (don't throw exceptions)
6. Idempotent where possible (safe to retry)
```

---

*Next week: Embeddings & Vector Search — we start building RAG (Retrieval-Augmented Generation). Claude will answer questions using YOUR documents, not just its training data.*
