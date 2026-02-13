# Week 11: MCP Concepts & Architecture

## 🎯 Learning Objectives

By the end of this week, you will:
- Understand what MCP is and why it exists (the "USB-C for AI" metaphor)
- Know the MCP architecture: hosts, clients, servers, and how they communicate
- Set up Claude Desktop as an MCP host
- Connect to pre-built MCP servers (filesystem, Git)
- Understand MCP primitives: tools, resources, and prompts
- Build your first "hello world" MCP server in Python

---

## 📖 Theory (20 min)

### The Problem MCP Solves

In Week 6, you gave Claude tools by defining them inline in your API call:

```python
tools = [{"name": "check_service", "description": "...", "input_schema": {...}}]
response = client.messages.create(..., tools=tools)
```

This works, but has problems at scale:
- **Duplication.** Every script that uses the same tools must define them again.
- **Tight coupling.** Your tools are hardcoded into your Python code. If you want Claude Desktop to use the same tools, you have to rebuild everything.
- **No standard.** Your tool format is different from someone else's. No way to share or reuse.

**MCP (Model Context Protocol)** solves this by creating a **standard protocol** for connecting AI models to external tools and data. Any MCP server works with any MCP client — just like any USB-C device works with any USB-C port.

### The USB-C Analogy

Before USB-C, every phone maker had a proprietary charging cable. You needed different cables for every device. USB-C standardized it — one cable works everywhere.

Before MCP, every AI integration was custom. You wrote bespoke code to connect Claude to Kubernetes, to your database, to Slack. MCP standardizes it — one protocol connects any AI to any tool.

```
Before MCP:                          After MCP:
Claude ←custom→ Kubernetes           Claude ←MCP→ Any MCP Server
Claude ←custom→ PostgreSQL                         ├── Kubernetes
Claude ←custom→ Slack                              ├── PostgreSQL
Claude ←custom→ Your tool                          ├── Slack
(each one is a unique integration)                 ├── Your custom tools
                                                   └── (reusable by anyone)
```

### MCP Architecture — Three Roles

```
┌──────────────────────────────┐
│  HOST (e.g., Claude Desktop) │
│  The AI application          │
│                              │
│  ┌────────────────────────┐  │
│  │  CLIENT                │  │     ┌─────────────────────┐
│  │  Talks MCP protocol    │──────→│  SERVER              │
│  │  (one per server)      │  │     │  Exposes tools/data  │
│  └────────────────────────┘  │     │  via MCP protocol    │
│                              │     └─────────────────────┘
│  ┌────────────────────────┐  │     ┌─────────────────────┐
│  │  CLIENT (another one)  │──────→│  SERVER (another)    │
│  └────────────────────────┘  │     └─────────────────────┘
└──────────────────────────────┘
```

**Host:** The AI application. Claude Desktop, Claude Code, your custom app. It contains the LLM and manages conversations.

**Client:** A connector inside the host that speaks MCP protocol. The host creates one client per server it connects to.

**Server:** An external process that exposes tools, resources, or prompts. It could be a Python script, a Node.js process, a Docker container — anything that speaks MCP.

### MCP Primitives — What Servers Expose

MCP servers can expose three types of things:

| Primitive | What It Is | Who Controls It | Example |
|-----------|-----------|----------------|---------|
| **Tools** | Functions the LLM can call | LLM decides when to call | `check_pod_status(namespace)` |
| **Resources** | Data the LLM can read | Application/user controls | `runbooks://redis-recovery` |
| **Prompts** | Reusable prompt templates | User selects | `incident-analysis` template |

**Tools** are what you built in Week 6 — but now exposed via a standard protocol.

**Resources** are like files or data — the LLM can read them for context. Think of your RAG runbooks as resources.

**Prompts** are reusable templates that the user can select. Your CRISP templates from Week 3 could be MCP prompts.

### Transport Layer — How They Talk

MCP supports two transport methods:

**stdio (standard I/O):** The host launches the server as a child process. They communicate via stdin/stdout. Simple, secure, local-only.

```
Claude Desktop → starts → python my_server.py
                  ↕ stdin/stdout (JSON-RPC messages)
```

**SSE (Server-Sent Events):** HTTP-based. The server runs as a web service. Used for remote servers.

```
Client → HTTP POST → http://my-server:3000/mcp
Server → SSE stream → back to client
```

For this week and next, we use **stdio** — it's simpler and how Claude Desktop works.

### JSON-RPC — The Message Format

MCP uses JSON-RPC 2.0 for messages. You don't need to write raw JSON-RPC — the SDK handles it. But understanding the format helps debugging:

```json
// Client asks: "What tools do you have?"
{"jsonrpc": "2.0", "method": "tools/list", "id": 1}

// Server responds:
{"jsonrpc": "2.0", "result": {"tools": [
  {"name": "check_pods", "description": "...", "inputSchema": {...}}
]}, "id": 1}

// Client says: "Call this tool"
{"jsonrpc": "2.0", "method": "tools/call", "params": {"name": "check_pods", "arguments": {"namespace": "payments"}}, "id": 2}

// Server responds with result
{"jsonrpc": "2.0", "result": {"content": [{"type": "text", "text": "3/5 pods running..."}]}, "id": 2}
```

---

## 🔨 Hands-On (50 min)

### Exercise 1: Your First MCP Server (25 min)

**Goal:** Build a minimal MCP server that exposes SRE tools, then connect it to Claude Desktop.

First, install the MCP Python SDK:

```bash
pip install mcp
```

Create `week11/sre_mcp_server.py`:

```python
"""
Week 11, Exercise 1: Your First MCP Server

This server exposes 3 SRE tools via MCP:
1. check_service_health — check if a service is healthy
2. get_pod_logs — get recent logs from a service
3. get_recent_deploys — check deployment history

When connected to Claude Desktop, Claude can call these tools
to investigate production issues.
"""
import json
from datetime import datetime
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import (
    Tool,
    TextContent,
)

# Create the MCP server
server = Server("sre-tools")


# ============================================================
# TOOL IMPLEMENTATIONS (simulated — in production these call real APIs)
# ============================================================

def check_service_health(service_name: str) -> dict:
    """Simulated health check."""
    services = {
        "payment-gateway": {
            "status": "degraded",
            "error_rate_percent": 4.7,
            "p99_latency_ms": 2100,
            "pods_running": 3,
            "pods_desired": 5,
            "restarts_last_hour": 47,
            "last_deployment": "v2.3.1 (15 minutes ago)",
            "active_alerts": ["ErrorRateHigh", "PodCrashLoopBackOff"]
        },
        "auth-service": {
            "status": "healthy",
            "error_rate_percent": 0.01,
            "p99_latency_ms": 42,
            "pods_running": 4,
            "pods_desired": 4,
            "restarts_last_hour": 0,
            "last_deployment": "v1.8.0 (3 days ago)",
            "active_alerts": []
        },
        "fraud-detection": {
            "status": "healthy",
            "error_rate_percent": 0.05,
            "p99_latency_ms": 230,
            "pods_running": 6,
            "pods_desired": 6,
            "restarts_last_hour": 0,
            "last_deployment": "v3.1.2 (1 day ago)",
            "active_alerts": []
        },
        "notification-service": {
            "status": "healthy",
            "error_rate_percent": 0.02,
            "p99_latency_ms": 55,
            "pods_running": 3,
            "pods_desired": 3,
            "restarts_last_hour": 0,
            "last_deployment": "v2.0.1 (5 days ago)",
            "active_alerts": []
        }
    }
    if service_name in services:
        return services[service_name]
    return {
        "error": f"Service '{service_name}' not found",
        "available_services": list(services.keys())
    }


def get_pod_logs(service_name: str, severity: str = "ERROR", lines: int = 10) -> dict:
    """Simulated pod log retrieval."""
    logs = {
        "payment-gateway": {
            "ERROR": [
                "14:23:01 ERROR [PaymentProcessor] java.lang.OutOfMemoryError: Java heap space",
                "14:23:01 ERROR [PaymentProcessor]   at BatchReconciliation.loadAllTransactions(BatchReconciliation.java:142)",
                "14:22:58 ERROR [PaymentProcessor] java.lang.OutOfMemoryError: Java heap space",
                "14:22:55 ERROR [PaymentProcessor] java.lang.OutOfMemoryError: Java heap space",
                "14:22:50 ERROR [HealthCheck] Liveness probe failed: OOM detected",
            ],
            "WARN": [
                "14:23:00 WARN [ConnectionPool] Active connections: 48/50 (96%)",
                "14:22:45 WARN [GC] Full GC triggered, heap at 98%",
                "14:22:30 WARN [BatchReconciliation] Loading 2.1M transactions into memory",
            ]
        },
        "auth-service": {
            "ERROR": [],
            "WARN": ["14:20:00 WARN [TokenRefresh] Token expires in 4 hours"]
        }
    }
    
    service_logs = logs.get(service_name, {"ERROR": [], "WARN": []})
    filtered = service_logs.get(severity, [])
    return {
        "service": service_name,
        "severity": severity,
        "lines": filtered[:lines],
        "count": len(filtered)
    }


def get_recent_deploys(service_name: str, hours_back: int = 24) -> dict:
    """Simulated deployment history."""
    deploys = {
        "payment-gateway": [
            {"version": "v2.3.1", "time": "14:00 today", "status": "completed",
             "changes": "Added BatchReconciliation for daily transaction reconciliation",
             "author": "ci-bot"},
            {"version": "v2.3.0", "time": "yesterday 10:00", "status": "completed",
             "changes": "Bug fix for card network timeout handling",
             "author": "ci-bot"},
        ],
        "auth-service": [
            {"version": "v1.8.0", "time": "3 days ago", "status": "completed",
             "changes": "Updated JWT library to v4.2", "author": "ci-bot"}
        ]
    }
    return {
        "service": service_name,
        "hours_back": hours_back,
        "deployments": deploys.get(service_name, []),
        "count": len(deploys.get(service_name, []))
    }


# ============================================================
# REGISTER TOOLS WITH MCP
# ============================================================

@server.list_tools()
async def list_tools() -> list[Tool]:
    """Tell MCP clients what tools this server offers."""
    return [
        Tool(
            name="check_service_health",
            description=(
                "Check the current health status of a production service. "
                "Returns status, error rate, latency, pod count, recent deploys, and active alerts. "
                "Available services: payment-gateway, auth-service, fraud-detection, notification-service. "
                "Use this as the FIRST step when investigating any service issue."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "service_name": {
                        "type": "string",
                        "description": "Name of the service to check",
                        "enum": ["payment-gateway", "auth-service", "fraud-detection", "notification-service"]
                    }
                },
                "required": ["service_name"]
            }
        ),
        Tool(
            name="get_pod_logs",
            description=(
                "Retrieve recent pod logs for a service. "
                "Returns log lines filtered by severity. "
                "Use when you need to see error messages, stack traces, or application behavior."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "service_name": {
                        "type": "string",
                        "description": "Service to get logs from"
                    },
                    "severity": {
                        "type": "string",
                        "enum": ["ERROR", "WARN", "ALL"],
                        "description": "Filter by log severity. Default: ERROR"
                    },
                    "lines": {
                        "type": "integer",
                        "description": "Max number of log lines. Default: 10",
                        "minimum": 1,
                        "maximum": 100
                    }
                },
                "required": ["service_name"]
            }
        ),
        Tool(
            name="get_recent_deploys",
            description=(
                "Get deployment history for a service within the last N hours. "
                "Use when investigating if a recent deployment caused an issue. "
                "Shows version, timestamp, changes, and deployment status."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "service_name": {
                        "type": "string",
                        "description": "Service to check deployments for"
                    },
                    "hours_back": {
                        "type": "integer",
                        "description": "How many hours of history. Default: 24",
                        "minimum": 1,
                        "maximum": 168
                    }
                },
                "required": ["service_name"]
            }
        )
    ]


@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    """Handle tool calls from the MCP client."""
    
    if name == "check_service_health":
        result = check_service_health(arguments["service_name"])
    elif name == "get_pod_logs":
        result = get_pod_logs(
            arguments["service_name"],
            arguments.get("severity", "ERROR"),
            arguments.get("lines", 10)
        )
    elif name == "get_recent_deploys":
        result = get_recent_deploys(
            arguments["service_name"],
            arguments.get("hours_back", 24)
        )
    else:
        result = {"error": f"Unknown tool: {name}"}
    
    return [TextContent(type="text", text=json.dumps(result, indent=2))]


# ============================================================
# RUN THE SERVER
# ============================================================

async def main():
    """Run the MCP server over stdio."""
    async with stdio_server() as (read_stream, write_stream):
        await server.run(read_stream, write_stream, server.create_initialization_options())

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

---

### Exercise 2: Connect to Claude Desktop (15 min)

**Goal:** Configure Claude Desktop to use your MCP server, so Claude can call your SRE tools in conversation.

**Step 1: Find your Claude Desktop config file**

```bash
# macOS
code ~/Library/Application\ Support/Claude/claude_desktop_config.json

# Windows
code %APPDATA%\Claude\claude_desktop_config.json

# Linux
code ~/.config/Claude/claude_desktop_config.json
```

**Step 2: Add your MCP server to the config**

Create or edit the config file:

```json
{
  "mcpServers": {
    "sre-tools": {
      "command": "python",
      "args": ["/full/path/to/week11/sre_mcp_server.py"],
      "env": {}
    }
  }
}
```

Replace `/full/path/to/` with the actual path on your system.

**Step 3: Restart Claude Desktop**

Completely quit and restart Claude Desktop. You should see a hammer 🔨 icon in the chat input area indicating tools are available.

**Step 4: Test it!**

Ask Claude these questions in Claude Desktop:

```
"Is the payment gateway healthy?"
→ Claude should call check_service_health("payment-gateway")

"Show me recent error logs from the payment gateway"
→ Claude should call get_pod_logs("payment-gateway", "ERROR")

"Were there any recent deployments to payment-gateway?"
→ Claude should call get_recent_deploys("payment-gateway")

"Investigate why the payment gateway seems degraded"
→ Claude should call MULTIPLE tools autonomously
```

Create `week11/test_desktop_queries.md` to document what you tested:

```markdown
# Claude Desktop + MCP Server Test Results

## Test 1: Simple health check
**Question:** "Is the payment gateway healthy?"
**Tools called:** check_service_health("payment-gateway")
**Claude's response:** [paste what Claude said]
**Did Claude use the tool data correctly?** Yes / No

## Test 2: Log retrieval
**Question:** "Show me recent error logs from payment-gateway"
**Tools called:** get_pod_logs("payment-gateway", "ERROR")
**Response quality:** [notes]

## Test 3: Deployment check
**Question:** "Any recent deploys to payment-gateway?"
**Tools called:** get_recent_deploys("payment-gateway")

## Test 4: Multi-tool investigation
**Question:** "Investigate why payment-gateway is degraded"
**Tools called:** [list all tools Claude called, in order]
**Did Claude chain tools correctly?** Yes / No
**Quality of final analysis:** [notes]
```

---

### Exercise 3: MCP Resources — Exposing Runbooks (10 min)

**Goal:** Add MCP resources to your server so Claude can read your SRE runbooks as context.

Add the following to `week11/sre_mcp_server.py` (or create a new file `week11/sre_mcp_resources.py`):

```python
"""
Week 11, Exercise 3: Adding MCP Resources

Resources are data that the LLM can read.
Unlike tools (which the LLM calls), resources are
selected by the USER or APPLICATION and provided as context.

Think: your runbooks as browsable documents.
"""
from mcp.types import Resource, TextContent

# ============================================================
# RUNBOOK DATA (same runbooks from Week 8)
# ============================================================

RUNBOOKS = {
    "pod-crashloop": {
        "title": "Pod CrashLoopBackOff Recovery",
        "content": """When a pod is in CrashLoopBackOff:
1. Check pod logs: kubectl logs <pod-name> --previous
2. Check events: kubectl describe pod <pod-name>
3. Common causes:
   - OOMKilled: increase memory limits
   - Image pull error: check registry credentials
   - App crash: check stack traces in logs
   - Liveness probe failure: adjust probe config
4. Quick restart: kubectl rollout restart deployment/<name>
5. Rollback: kubectl rollout undo deployment/<name>"""
    },
    "redis-recovery": {
        "title": "Redis Cache Recovery",
        "content": """When Redis is failing:
1. Test: redis-cli -h <host> ping (expect PONG)
2. Memory: redis-cli info memory
3. If full: redis-cli CONFIG GET maxmemory-policy (should be allkeys-lru)
4. Clients: redis-cli info clients
5. Cluster: redis-cli cluster info (expect cluster_state:ok)
6. Emergency: redis-cli FLUSHALL (if data loss acceptable)"""
    },
    "db-connections": {
        "title": "PostgreSQL Connection Issues",
        "content": """When database connections are exhausted:
1. Check: SELECT count(*), state FROM pg_stat_activity GROUP BY state;
2. Find idle: SELECT pid, state, query_start FROM pg_stat_activity WHERE state='idle' ORDER BY query_start;
3. Kill idle: SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state='idle' AND query_start < now() - interval '10 min';
4. Check limit: SHOW max_connections;
5. Long-term: deploy pgbouncer as connection pooler"""
    },
}


# Add these handlers to your server:

@server.list_resources()
async def list_resources() -> list[Resource]:
    """List available runbooks as MCP resources."""
    resources = []
    for key, rb in RUNBOOKS.items():
        resources.append(
            Resource(
                uri=f"runbook://{key}",
                name=rb["title"],
                description=f"SRE runbook: {rb['title']}",
                mimeType="text/plain"
            )
        )
    return resources


@server.read_resource()
async def read_resource(uri: str) -> str:
    """Read a specific runbook by URI."""
    # Parse: "runbook://pod-crashloop" → "pod-crashloop"
    key = uri.replace("runbook://", "")
    
    if key in RUNBOOKS:
        rb = RUNBOOKS[key]
        return f"# {rb['title']}\n\n{rb['content']}"
    
    raise ValueError(f"Runbook not found: {key}")
```

**How resources differ from tools:**
- **Tools:** Claude decides when to call them. "I need to check pod status" → calls tool.
- **Resources:** The user or application selects them. "Read the Redis runbook" → resource loaded into context.

In Claude Desktop, resources appear in the attachment menu (📎) — the user can manually attach a runbook to the conversation.

---

## 📝 Drills (20 min)

### Drill 1: Add MCP Prompts

Add reusable prompt templates to your server:

```python
@server.list_prompts()
async def list_prompts() -> list[Prompt]:
    return [
        Prompt(
            name="incident-analysis",
            description="Structured incident analysis template",
            arguments=[
                PromptArgument(name="service", description="Affected service", required=True),
                PromptArgument(name="symptoms", description="What's happening", required=True),
            ]
        ),
        Prompt(
            name="change-review",
            description="Review a proposed production change",
            arguments=[
                PromptArgument(name="change", description="What's being changed", required=True),
            ]
        )
    ]

@server.get_prompt()
async def get_prompt(name: str, arguments: dict) -> GetPromptResult:
    if name == "incident-analysis":
        return GetPromptResult(
            messages=[{
                "role": "user",
                "content": f"Analyze this incident using the SRE tools available:\n\nService: {arguments['service']}\nSymptoms: {arguments['symptoms']}\n\nUse check_service_health, get_pod_logs, and get_recent_deploys to investigate before providing your analysis."
            }]
        )
```

Prompts appear as slash commands in Claude Desktop — the user types `/incident-analysis` to use the template.

### Drill 2: Add Error Handling and Logging

Add robust error handling to your MCP server:

```python
import logging
logging.basicConfig(level=logging.INFO, filename="mcp_server.log")

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    logging.info(f"Tool called: {name}({arguments})")
    try:
        result = ...
        logging.info(f"Tool result: {name} → success")
        return [TextContent(type="text", text=json.dumps(result, indent=2))]
    except Exception as e:
        logging.error(f"Tool error: {name} → {e}")
        return [TextContent(type="text", text=json.dumps({"error": str(e)}))]
```

Check `mcp_server.log` after testing to see what Claude called and when.

### Drill 3: Connect a Pre-Built MCP Server

Install and connect the official Filesystem MCP server:

```bash
npm install -g @modelcontextprotocol/server-filesystem
```

Add to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "sre-tools": { ... },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/your/project"]
    }
  }
}
```

Now Claude Desktop can read and write files on your computer. Test: "List the files in my project directory."

### Drill 4: Architecture Diagram

Create `week11/drill4_architecture.md`:

Draw (in ASCII or markdown) the MCP architecture for your SRE system:

```
Claude Desktop (Host)
  ├── MCP Client → sre-tools server (your Python server)
  │                  ├── check_service_health
  │                  ├── get_pod_logs
  │                  └── get_recent_deploys
  │
  ├── MCP Client → filesystem server (read runbook files)
  │
  └── MCP Client → [future: kubernetes server]
```

Plan what servers you'd build for your Visa environment:
- ClickHouse log query server
- Kubernetes management server
- PagerDuty integration server
- Confluence/wiki search server

---

## ✅ Week 11 Checklist

Before moving to Week 12:

- [ ] Can explain MCP: hosts, clients, servers, and how they communicate
- [ ] Can explain the three MCP primitives: tools, resources, prompts
- [ ] Have a working MCP server in Python with 3+ tools
- [ ] Server connected to Claude Desktop (or understand the config process)
- [ ] Understand stdio transport (how host launches and talks to server)
- [ ] Know the difference between tools (LLM-controlled) and resources (user-controlled)
- [ ] Understand why MCP > inline tool definitions for production

---

## 🧠 Key Concepts from This Week

### MCP vs Inline Tool Use

```
Inline (Week 6):
  + Simple, all in one file
  + Good for scripts and prototyping
  - Tools hardcoded in your code
  - Can't reuse across apps
  - No standard format

MCP (Week 11+):
  + Standard protocol — reusable across any MCP host
  + Tools, resources, and prompts in one server
  + Claude Desktop, Claude Code, custom apps all use the same servers
  + Community of pre-built servers (filesystem, git, slack, etc.)
  - More setup than inline
  - Needs server process management
```

### When to Use MCP vs Inline Tools

```
Use Inline Tools:
  - Quick scripts and experiments
  - One-off automation
  - CI/CD pipelines
  - Batch processing

Use MCP Servers:
  - Tools you want to reuse across multiple apps
  - Claude Desktop integration
  - Team-shared tools
  - Complex multi-tool systems
  - Production tool platforms
```

### MCP Server Template (memorize this pattern)

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

server = Server("my-server")

@server.list_tools()
async def list_tools() -> list[Tool]:
    return [Tool(name="...", description="...", inputSchema={...})]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    result = do_something(**arguments)
    return [TextContent(type="text", text=json.dumps(result))]

async def main():
    async with stdio_server() as (read, write):
        await server.run(read, write, server.create_initialization_options())

asyncio.run(main())
```

---

*Next week: Building Custom MCP Servers — you'll build production-quality MCP servers that connect to Kubernetes, ClickHouse, and your runbook knowledge base (RAG as a resource!).*
