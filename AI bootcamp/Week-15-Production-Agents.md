# Week 15: Production Agents

## 🎯 Learning Objectives

By the end of this week, you will:
- Build agents with persistent memory that learn from past investigations
- Implement robust error recovery — agents that adapt when tools fail
- Add full observability: structured logging, tracing, metrics dashboards
- Create the `sre-agent` portfolio project — a production-ready investigation agent
- Handle real-world agent challenges: hallucination detection, stuck loops, partial failures
- Know when to use agents vs simpler tool-use scripts

---

## 📖 Theory (20 min)

### The Gap Between Demo Agent and Production Agent

Your Week 14 agent works in demos. Production has real challenges:

| Demo Agent | Production Agent |
|-----------|-----------------|
| Tools always work | Tools fail (network timeout, API errors, rate limits) |
| Clean simulated data | Messy, incomplete, contradictory real data |
| One investigation at a time | Multiple concurrent investigations |
| No memory between runs | Learns from past incidents ("last time this happened...") |
| No observability | Full trace of every decision and action |
| User watches the output | Runs autonomously, reports results via Slack/PagerDuty |

### Persistent Memory — Learning from History

Your Week 14 agent forgets everything after each run. A production agent should remember:

```
Investigation #1 (Tuesday):
  "payment-gateway degraded → OOMKilled → BatchReconciliation loading too much data"
  Fix: increased memory to 2Gi

Investigation #2 (Friday):
  "payment-gateway degraded → ?"
  Agent recalls: "Last time this was OOMKilled from BatchReconciliation. 
                  Let me check if the same pattern is happening."
```

This is **episodic memory** — the agent stores summaries of past investigations and retrieves relevant ones when starting a new task.

### Error Recovery Patterns

| Tool Failure | Recovery Strategy |
|---|---|
| Timeout | Retry with backoff (1s, 2s, 4s) |
| 500 Server Error | Retry once, then try alternative tool |
| Permission Denied | Log warning, skip this data source, note the gap |
| Invalid Response | Parse what you can, flag uncertainty |
| Tool Not Available | Adapt plan — investigate with remaining tools |

The key principle: **degrade gracefully, never crash.** A partial investigation is better than no investigation.

### Agent Observability

Every production agent needs:

```
STRUCTURED LOGS (for debugging):
  2024-10-15 14:23:01 | AGENT_START | task="investigate payment-gateway"
  2024-10-15 14:23:02 | TOOL_CALL  | tool=check_service_health | args={...}
  2024-10-15 14:23:03 | TOOL_RESULT| tool=check_service_health | latency=1.2s | status=success
  2024-10-15 14:23:03 | REASONING  | "Service is degraded, checking pods..."
  2024-10-15 14:23:15 | AGENT_DONE | iterations=5 | cost=$0.03 | duration=14s

METRICS (for dashboards):
  agent_investigations_total: 47
  agent_avg_iterations: 4.2
  agent_avg_cost_usd: 0.028
  agent_avg_duration_seconds: 12.5
  agent_tool_failures_total: 3
  agent_root_cause_found_rate: 0.89

TRACES (for deep debugging):
  Full conversation history stored per investigation
  Can replay any investigation to understand agent behavior
```

---

## 🔨 Hands-On (50 min)

### Exercise 1: Agent with Persistent Memory (20 min)

**Goal:** Build an agent that stores investigation summaries and retrieves relevant past cases when starting a new investigation.

Create `week15/persistent_agent.py`:

```python
"""
Week 15, Exercise 1: Agent with Persistent Memory

The agent remembers past investigations and uses them to:
- Recognize recurring issues faster
- Reference what fixed the problem last time
- Build institutional knowledge automatically
"""
import anthropic
import json
import time
import chromadb
from sentence_transformers import SentenceTransformer
from dataclasses import dataclass, field
from typing import Optional

client = anthropic.Anthropic()
embedder = SentenceTransformer('all-MiniLM-L6-v2')


# ============================================================
# EPISODIC MEMORY — Stores past investigation summaries
# ============================================================

class EpisodicMemory:
    """
    Stores summaries of past investigations in a vector database.
    When a new investigation starts, retrieves similar past cases.
    """
    
    def __init__(self, persist_path: str = None):
        self.chroma = chromadb.Client()
        self.collection = self.chroma.get_or_create_collection("agent_episodes")
        self.episode_count = 0
    
    def store_episode(self, task: str, findings: list[str], root_cause: str,
                      fix_applied: str, severity: str, duration_seconds: float):
        """Store a completed investigation as an episode."""
        summary = f"Task: {task}\nRoot Cause: {root_cause}\nFix: {fix_applied}\nSeverity: {severity}"
        
        embedding = embedder.encode(summary).tolist()
        episode_id = f"ep-{self.episode_count:04d}"
        
        self.collection.add(
            ids=[episode_id],
            documents=[summary],
            metadatas=[{
                "task": task[:200],
                "root_cause": root_cause[:200],
                "fix": fix_applied[:200],
                "severity": severity,
                "duration_s": duration_seconds,
                "timestamp": time.strftime("%Y-%m-%d %H:%M:%S"),
                "findings_count": len(findings),
            }],
            embeddings=[embedding]
        )
        
        self.episode_count += 1
        return episode_id
    
    def recall_similar(self, task: str, n_results: int = 3) -> list[dict]:
        """Find past investigations similar to the current task."""
        if self.collection.count() == 0:
            return []
        
        query_emb = embedder.encode(task).tolist()
        results = self.collection.query(
            query_embeddings=[query_emb],
            n_results=min(n_results, self.collection.count()),
            include=["documents", "metadatas", "distances"]
        )
        
        episodes = []
        for i in range(len(results['ids'][0])):
            if results['distances'][0][i] < 1.5:  # Relevance threshold
                episodes.append({
                    "id": results['ids'][0][i],
                    "summary": results['documents'][0][i],
                    "metadata": results['metadatas'][0][i],
                    "relevance": round(1 - results['distances'][0][i] / 2, 3)
                })
        
        return episodes
    
    def get_stats(self) -> dict:
        return {
            "total_episodes": self.collection.count(),
            "memory_size": self.episode_count
        }


# ============================================================
# SIMULATED TOOLS (compact version)
# ============================================================

def execute_tool(name: str, args: dict) -> dict:
    """Simulated SRE tools."""
    data = {
        "check_service_health": {
            "payment-gateway": {"status": "degraded", "error_rate": 4.7, "pods": "3/5", "alerts": ["ErrorRateHigh", "OOMKilled"]},
            "auth-service": {"status": "healthy", "error_rate": 0.01, "pods": "4/4", "alerts": []},
            "fraud-detection": {"status": "healthy", "error_rate": 0.1, "pods": "6/6", "alerts": []},
        },
        "list_pods": {
            "payments": {"pods": [
                {"name": "payment-gw-def34", "status": "CrashLoopBackOff", "restarts": 47},
                {"name": "payment-gw-ghi56", "status": "CrashLoopBackOff", "restarts": 32},
                {"name": "payment-gw-abc12", "status": "Running", "restarts": 0},
            ]}
        },
        "get_pod_logs": {
            "payment-gw-def34": {"logs": [
                "WARN  Loading 2.1M transactions into memory",
                "ERROR java.lang.OutOfMemoryError: Java heap space",
                "ERROR   at BatchReconciliation.loadAllTransactions:142",
            ]}
        },
        "describe_pod": {
            "payment-gw-def34": {"status": "CrashLoopBackOff", "last_state": "OOMKilled", "memory_limit": "512Mi", "restarts": 47}
        },
        "get_recent_deploys": {
            "payment-gateway": {"deploys": [{"version": "v2.3.1", "time": "today 14:00", "changes": "Added BatchReconciliation"}]}
        },
        "search_runbooks": {"default": {"results": [{"title": "OOMKilled Recovery", "content": "Increase memory limits, refactor batch jobs to use streaming"}]}}
    }
    
    if name == "check_service_health":
        return data["check_service_health"].get(args.get("service_name", ""), {"error": "Unknown service"})
    elif name == "list_pods":
        return data["list_pods"].get(args.get("namespace", ""), {"pods": []})
    elif name == "get_pod_logs":
        return data["get_pod_logs"].get(args.get("pod_name", ""), {"logs": ["No logs"]})
    elif name == "describe_pod":
        return data["describe_pod"].get(args.get("pod_name", ""), {"error": "Not found"})
    elif name == "get_recent_deploys":
        return data["get_recent_deploys"].get(args.get("service_name", ""), {"deploys": []})
    elif name == "search_runbooks":
        return data["search_runbooks"]["default"]
    return {"error": f"Unknown: {name}"}


TOOLS = [
    {"name": "check_service_health", "description": "Check service health status, error rate, alerts.",
     "input_schema": {"type": "object", "properties": {"service_name": {"type": "string"}}, "required": ["service_name"]}},
    {"name": "list_pods", "description": "List pods with status and restarts.",
     "input_schema": {"type": "object", "properties": {"namespace": {"type": "string"}}, "required": ["namespace"]}},
    {"name": "get_pod_logs", "description": "Get error logs from a pod.",
     "input_schema": {"type": "object", "properties": {"pod_name": {"type": "string"}}, "required": ["pod_name"]}},
    {"name": "describe_pod", "description": "Get pod details: resources, limits, termination reason.",
     "input_schema": {"type": "object", "properties": {"pod_name": {"type": "string"}}, "required": ["pod_name"]}},
    {"name": "get_recent_deploys", "description": "Recent deployment history.",
     "input_schema": {"type": "object", "properties": {"service_name": {"type": "string"}}, "required": ["service_name"]}},
    {"name": "search_runbooks", "description": "Search SRE runbooks for procedures.",
     "input_schema": {"type": "object", "properties": {"query": {"type": "string"}}, "required": ["query"]}},
]


# ============================================================
# PRODUCTION AGENT WITH MEMORY
# ============================================================

class ProductionSREAgent:
    """
    Production-ready SRE investigation agent with:
    - Episodic memory (learns from past investigations)
    - Error recovery (handles tool failures)
    - Structured observability (logs every action)
    - Cost tracking
    """
    
    def __init__(self, max_iterations: int = 10, max_cost: float = 0.50):
        self.max_iterations = max_iterations
        self.max_cost = max_cost
        self.memory = EpisodicMemory()
        self.investigation_log = []
    
    def _log(self, event: str, **kwargs):
        entry = {"time": time.strftime("%H:%M:%S"), "event": event, **kwargs}
        self.investigation_log.append(entry)
        
        # Pretty print
        detail = " | ".join(f"{k}={v}" for k, v in kwargs.items() if k != "time")
        print(f"  [{entry['time']}] {event:20s} | {detail[:100]}")
    
    def investigate(self, task: str) -> dict:
        """Run a full investigation with memory recall and structured logging."""
        start = time.time()
        self.investigation_log = []
        total_cost = 0.0
        
        self._log("AGENT_START", task=task[:80])
        
        # ---- RECALL PAST EPISODES ----
        past_cases = self.memory.recall_similar(task)
        memory_context = ""
        
        if past_cases:
            self._log("MEMORY_RECALL", episodes_found=len(past_cases))
            memory_parts = []
            for ep in past_cases:
                memory_parts.append(
                    f"[Past case, relevance={ep['relevance']}]\n"
                    f"{ep['summary']}\n"
                    f"Duration: {ep['metadata']['duration_s']}s"
                )
            memory_context = "\n\n".join(memory_parts)
        else:
            self._log("MEMORY_RECALL", episodes_found=0)
        
        # ---- BUILD SYSTEM PROMPT ----
        system = f"""You are an expert SRE agent investigating production issues autonomously.

INVESTIGATION RULES:
1. Start with check_service_health
2. Drill into unhealthy pods (logs, describe)
3. Check recent deployments
4. Search runbooks for fixes
5. Stop when you have root cause evidence

FINAL ANSWER FORMAT:
SEVERITY: P1/P2/P3/P4
ROOT CAUSE: [specific cause with evidence]
EVIDENCE: [key data points]
IMMEDIATE ACTIONS: [numbered commands]
LONG-TERM FIX: [prevention]
"""
        
        if memory_context:
            system += f"""
PAST SIMILAR INVESTIGATIONS (use these to guide your investigation):
{memory_context}

If a past case is very similar, check if the same root cause applies. Reference it in your analysis.
"""
        
        messages = [{"role": "user", "content": task}]
        
        # ---- AGENTIC LOOP ----
        for iteration in range(self.max_iterations):
            if total_cost >= self.max_cost:
                self._log("GUARDRAIL", reason="cost_limit", cost=total_cost)
                break
            
            try:
                response = client.messages.create(
                    model="claude-sonnet-4-5-20250514",
                    max_tokens=2000,
                    temperature=0,
                    system=system,
                    tools=TOOLS,
                    messages=messages
                )
            except Exception as e:
                self._log("API_ERROR", error=str(e))
                time.sleep(2)  # Backoff
                continue
            
            # Track cost
            tokens = response.usage.input_tokens + response.usage.output_tokens
            cost = (response.usage.input_tokens * 3 + response.usage.output_tokens * 15) / 1_000_000
            total_cost += cost
            
            # ---- DONE? ----
            if response.stop_reason == "end_turn":
                answer = "".join(b.text for b in response.content if b.type == "text")
                duration = time.time() - start
                
                self._log("AGENT_DONE", iterations=iteration+1, 
                          cost=f"${total_cost:.4f}", duration=f"{duration:.1f}s")
                
                # ---- STORE EPISODE ----
                # Extract root cause from the answer
                root_cause = ""
                fix = ""
                severity = "P3"
                for line in answer.split("\n"):
                    if line.strip().startswith("ROOT CAUSE:"):
                        root_cause = line.split("ROOT CAUSE:")[1].strip()
                    elif line.strip().startswith("SEVERITY:"):
                        severity = line.split("SEVERITY:")[1].strip()
                    elif line.strip().startswith("LONG-TERM FIX:") or line.strip().startswith("IMMEDIATE ACTIONS:"):
                        fix += line.split(":", 1)[1].strip() + " "
                
                episode_id = self.memory.store_episode(
                    task=task,
                    findings=[entry.get("detail", "") for entry in self.investigation_log if entry["event"] == "TOOL_RESULT"],
                    root_cause=root_cause or "Unknown",
                    fix_applied=fix or "See investigation",
                    severity=severity,
                    duration_seconds=duration
                )
                self._log("MEMORY_STORED", episode_id=episode_id)
                
                return {
                    "answer": answer,
                    "iterations": iteration + 1,
                    "cost": total_cost,
                    "duration": duration,
                    "past_cases_used": len(past_cases),
                    "episode_id": episode_id,
                }
            
            # ---- EXECUTE TOOLS ----
            if response.stop_reason == "tool_use":
                tool_results = []
                
                for block in response.content:
                    if block.type == "text" and block.text.strip():
                        self._log("REASONING", thought=block.text.strip()[:100])
                    
                    if block.type == "tool_use":
                        self._log("TOOL_CALL", tool=block.name, args=json.dumps(block.input)[:80])
                        
                        try:
                            tool_start = time.time()
                            result = execute_tool(block.name, block.input)
                            tool_latency = time.time() - tool_start
                            
                            self._log("TOOL_RESULT", tool=block.name, 
                                      latency=f"{tool_latency*1000:.0f}ms", status="success")
                            
                            tool_results.append({
                                "type": "tool_result",
                                "tool_use_id": block.id,
                                "content": json.dumps(result)
                            })
                        
                        except Exception as e:
                            self._log("TOOL_ERROR", tool=block.name, error=str(e))
                            tool_results.append({
                                "type": "tool_result",
                                "tool_use_id": block.id,
                                "content": json.dumps({
                                    "error": f"Tool failed: {str(e)}",
                                    "suggestion": "Try an alternative approach or skip this data source."
                                }),
                                "is_error": True
                            })
                
                messages.append({"role": "assistant", "content": response.content})
                messages.append({"role": "user", "content": tool_results})
        
        duration = time.time() - start
        self._log("AGENT_INCOMPLETE", reason="max_iterations")
        return {"answer": "Investigation incomplete", "iterations": self.max_iterations,
                "cost": total_cost, "duration": duration}


# ============================================================
# RUN: Multiple investigations — agent learns over time
# ============================================================

if __name__ == "__main__":
    agent = ProductionSREAgent(max_iterations=10, max_cost=0.50)
    
    # ---- Investigation 1: First time seeing this issue ----
    print(f"\n{'🔍' * 35}")
    print("INVESTIGATION #1 (first time)")
    print(f"{'🔍' * 35}\n")
    
    result1 = agent.investigate(
        "Payment gateway is degraded. Error rate spiking. Customers reporting failed transactions."
    )
    print(f"\n📊 Result: {result1['iterations']} iterations, ${result1['cost']:.4f}, {result1['duration']:.1f}s")
    print(f"   Past cases used: {result1['past_cases_used']}")
    print(f"   Memory: {agent.memory.get_stats()}")
    
    # ---- Investigation 2: Similar issue — agent should recall past case ----
    print(f"\n\n{'🔍' * 35}")
    print("INVESTIGATION #2 (similar issue — should recall past case)")
    print(f"{'🔍' * 35}\n")
    
    result2 = agent.investigate(
        "Payment-gateway seeing high error rates again. Pods are restarting frequently."
    )
    print(f"\n📊 Result: {result2['iterations']} iterations, ${result2['cost']:.4f}, {result2['duration']:.1f}s")
    print(f"   Past cases used: {result2['past_cases_used']}")
    print(f"   Memory: {agent.memory.get_stats()}")
    
    # ---- Investigation 3: Different issue ----
    print(f"\n\n{'🔍' * 35}")
    print("INVESTIGATION #3 (different issue)")
    print(f"{'🔍' * 35}\n")
    
    result3 = agent.investigate(
        "Auth service seems slow. JWT validation taking too long. Not sure if it's the service or database."
    )
    print(f"\n📊 Result: {result3['iterations']} iterations, ${result3['cost']:.4f}, {result3['duration']:.1f}s")
    print(f"   Past cases used: {result3['past_cases_used']}")
    print(f"   Memory: {agent.memory.get_stats()}")
```

---

### Exercise 2: Error Recovery Agent (15 min)

**Goal:** Build an agent that handles tool failures gracefully — retries, alternatives, and partial results.

Create `week15/error_recovery.py`:

```python
"""
Week 15, Exercise 2: Error Recovery Patterns

Tools fail in production. The agent must handle:
- Timeouts (retry with backoff)
- Permission errors (skip and note the gap)
- Invalid data (parse what you can)
- Tool unavailability (adapt the plan)
"""
import json
import time
import random


class ResilientToolExecutor:
    """
    Wraps tool execution with retry logic, error handling,
    and alternative strategies.
    """
    
    def __init__(self, max_retries: int = 2, base_delay: float = 1.0):
        self.max_retries = max_retries
        self.base_delay = base_delay
        self.failure_log = []
        self.success_count = 0
        self.failure_count = 0
    
    def execute_with_retry(self, tool_name: str, args: dict, 
                           tool_func: callable) -> dict:
        """
        Execute a tool with retry logic and error handling.
        Returns result dict — always includes a 'status' field.
        """
        last_error = None
        
        for attempt in range(self.max_retries + 1):
            try:
                result = tool_func(tool_name, args)
                
                # Validate the result
                if isinstance(result, dict) and "error" in result:
                    # Tool returned an error (not an exception)
                    error_msg = result["error"]
                    
                    if "permission" in error_msg.lower() or "forbidden" in error_msg.lower():
                        # Permission errors won't fix with retry
                        self.failure_count += 1
                        self.failure_log.append({
                            "tool": tool_name, "error": error_msg,
                            "recovery": "skipped_permission"
                        })
                        return {
                            "status": "partial",
                            "data": None,
                            "error": error_msg,
                            "note": f"Skipped {tool_name} due to permissions. Investigation continues without this data.",
                            "recovery_action": "skipped"
                        }
                    
                    if attempt < self.max_retries:
                        delay = self.base_delay * (2 ** attempt)
                        time.sleep(delay)
                        continue
                    
                    # All retries failed
                    self.failure_count += 1
                    return {
                        "status": "failed",
                        "data": None,
                        "error": error_msg,
                        "attempts": attempt + 1,
                        "note": f"Tool {tool_name} failed after {attempt+1} attempts."
                    }
                
                # Success
                self.success_count += 1
                return {"status": "success", "data": result}
            
            except TimeoutError as e:
                last_error = str(e)
                if attempt < self.max_retries:
                    delay = self.base_delay * (2 ** attempt)
                    time.sleep(delay)
                    continue
            
            except Exception as e:
                last_error = str(e)
                self.failure_count += 1
                self.failure_log.append({
                    "tool": tool_name, "error": str(e),
                    "recovery": "exception"
                })
                return {
                    "status": "error",
                    "data": None,
                    "error": str(e),
                    "note": f"Unexpected error in {tool_name}. Skipping this data source."
                }
        
        self.failure_count += 1
        return {
            "status": "failed",
            "data": None,
            "error": last_error,
            "attempts": self.max_retries + 1
        }
    
    def get_reliability_stats(self) -> dict:
        total = self.success_count + self.failure_count
        return {
            "total_calls": total,
            "successes": self.success_count,
            "failures": self.failure_count,
            "success_rate": f"{self.success_count/total:.1%}" if total > 0 else "N/A",
            "failure_log": self.failure_log
        }


# ============================================================
# SIMULATED FLAKY TOOLS (randomly fail to test recovery)
# ============================================================

def flaky_tool_executor(name: str, args: dict) -> dict:
    """Tools that sometimes fail — simulates production reality."""
    
    # 20% chance of timeout
    if random.random() < 0.2:
        raise TimeoutError(f"{name} timed out after 30s")
    
    # 10% chance of permission error
    if random.random() < 0.1:
        return {"error": f"Permission denied: cannot access {name}"}
    
    # Normal execution
    simple_data = {
        "check_service_health": {"status": "degraded", "error_rate": 4.7},
        "list_pods": {"pods": [{"name": "pod-1", "status": "CrashLoopBackOff"}]},
        "get_pod_logs": {"logs": ["ERROR OutOfMemoryError"]},
    }
    
    return simple_data.get(name, {"result": f"{name} completed"})


# ============================================================
# TEST ERROR RECOVERY
# ============================================================

if __name__ == "__main__":
    executor = ResilientToolExecutor(max_retries=2, base_delay=0.5)
    
    print("=" * 70)
    print("ERROR RECOVERY TEST (running 20 tool calls with random failures)")
    print("=" * 70)
    
    random.seed(42)  # Reproducible
    
    tools_to_test = [
        ("check_service_health", {"service_name": "payment-gateway"}),
        ("list_pods", {"namespace": "payments"}),
        ("get_pod_logs", {"pod_name": "pod-1"}),
        ("check_service_health", {"service_name": "auth-service"}),
        ("list_pods", {"namespace": "auth"}),
    ] * 4  # 20 calls total
    
    for tool_name, args in tools_to_test:
        result = executor.execute_with_retry(tool_name, args, flaky_tool_executor)
        status_emoji = {"success": "✅", "partial": "🟡", "failed": "❌", "error": "💥"}
        emoji = status_emoji.get(result["status"], "❓")
        
        detail = ""
        if result["status"] != "success":
            detail = f" — {result.get('note', result.get('error', ''))[:60]}"
        
        print(f"  {emoji} {tool_name:25s} → {result['status']}{detail}")
    
    print(f"\n📊 Reliability stats: {json.dumps(executor.get_reliability_stats(), indent=2)}")
```

---

### Exercise 3: Full Agent Observability (15 min)

**Goal:** Add structured logging, metrics collection, and investigation tracing for production monitoring.

Create `week15/agent_observability.py`:

```python
"""
Week 15, Exercise 3: Agent Observability

Production agents need:
1. Structured event log (every decision, action, observation)
2. Metrics (aggregated stats for dashboards)
3. Investigation trace (replayable record)
4. Alerts (detect stuck or expensive agents)
"""
import json
import time
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class AgentEvent:
    """A single event in the agent's lifecycle."""
    timestamp: float
    event_type: str  # PLAN, REASON, TOOL_CALL, TOOL_RESULT, ERROR, DECISION, DONE
    data: dict = field(default_factory=dict)
    
    def to_dict(self):
        return {
            "time": time.strftime("%H:%M:%S", time.localtime(self.timestamp)),
            "offset_s": 0,  # Set later relative to start
            "type": self.event_type,
            **self.data
        }


class AgentObserver:
    """
    Observability system for agent investigations.
    
    Collects events, computes metrics, detects anomalies.
    """
    
    def __init__(self, investigation_id: str = None):
        self.id = investigation_id or f"inv-{int(time.time())}"
        self.events: list[AgentEvent] = []
        self.start_time = time.time()
        self.metrics = {
            "tool_calls": 0,
            "tool_successes": 0,
            "tool_failures": 0,
            "total_tokens": 0,
            "total_cost_usd": 0.0,
            "iterations": 0,
            "reasoning_steps": 0,
        }
        self.tool_latencies = {}  # tool_name → [latencies]
    
    def emit(self, event_type: str, **kwargs):
        """Record an event."""
        event = AgentEvent(
            timestamp=time.time(),
            event_type=event_type,
            data=kwargs
        )
        self.events.append(event)
        
        # Update metrics
        if event_type == "TOOL_CALL":
            self.metrics["tool_calls"] += 1
        elif event_type == "TOOL_RESULT":
            if kwargs.get("status") == "success":
                self.metrics["tool_successes"] += 1
            else:
                self.metrics["tool_failures"] += 1
            
            # Track latency per tool
            tool = kwargs.get("tool", "unknown")
            latency = kwargs.get("latency_ms", 0)
            self.tool_latencies.setdefault(tool, []).append(latency)
        
        elif event_type == "REASONING":
            self.metrics["reasoning_steps"] += 1
        elif event_type == "LLM_CALL":
            self.metrics["iterations"] += 1
            self.metrics["total_tokens"] += kwargs.get("tokens", 0)
            self.metrics["total_cost_usd"] += kwargs.get("cost", 0)
    
    def check_anomalies(self) -> list[str]:
        """Detect potential problems with the agent's behavior."""
        anomalies = []
        elapsed = time.time() - self.start_time
        
        if elapsed > 60:
            anomalies.append(f"Investigation taking too long: {elapsed:.0f}s")
        
        if self.metrics["iterations"] > 8:
            anomalies.append(f"Too many iterations: {self.metrics['iterations']}")
        
        if self.metrics["total_cost_usd"] > 0.30:
            anomalies.append(f"High cost: ${self.metrics['total_cost_usd']:.3f}")
        
        if self.metrics["tool_failures"] > 3:
            anomalies.append(f"High tool failure rate: {self.metrics['tool_failures']} failures")
        
        # Detect loops — same tool called 3+ times
        tool_counts = {}
        for e in self.events:
            if e.event_type == "TOOL_CALL":
                tool = e.data.get("tool", "")
                call_key = f"{tool}({json.dumps(e.data.get('args', {}), sort_keys=True)})"
                tool_counts[call_key] = tool_counts.get(call_key, 0) + 1
        
        for call_key, count in tool_counts.items():
            if count >= 3:
                anomalies.append(f"Possible loop: {call_key} called {count} times")
        
        return anomalies
    
    def get_timeline(self) -> list[dict]:
        """Get a readable timeline of the investigation."""
        timeline = []
        for event in self.events:
            entry = event.to_dict()
            entry["offset_s"] = round(event.timestamp - self.start_time, 2)
            timeline.append(entry)
        return timeline
    
    def get_metrics(self) -> dict:
        """Get aggregated metrics for dashboards."""
        elapsed = time.time() - self.start_time
        
        avg_latencies = {}
        for tool, lats in self.tool_latencies.items():
            avg_latencies[tool] = {
                "avg_ms": round(sum(lats) / len(lats), 1),
                "max_ms": round(max(lats), 1),
                "calls": len(lats)
            }
        
        return {
            "investigation_id": self.id,
            "duration_s": round(elapsed, 1),
            **self.metrics,
            "tool_success_rate": (
                f"{self.metrics['tool_successes']/(self.metrics['tool_calls'] or 1):.0%}"
            ),
            "avg_tokens_per_iteration": (
                round(self.metrics["total_tokens"] / max(self.metrics["iterations"], 1))
            ),
            "tool_latencies": avg_latencies,
            "anomalies": self.check_anomalies(),
        }
    
    def get_trace(self) -> dict:
        """Get the full replayable trace."""
        return {
            "investigation_id": self.id,
            "start_time": time.strftime("%Y-%m-%d %H:%M:%S", time.localtime(self.start_time)),
            "duration_s": round(time.time() - self.start_time, 1),
            "events": self.get_timeline(),
            "metrics": self.get_metrics(),
        }


# ============================================================
# DEMO: Observer in action
# ============================================================

if __name__ == "__main__":
    observer = AgentObserver("inv-demo-001")
    
    # Simulate an investigation
    observer.emit("PLAN", steps=["check health", "check pods", "get logs", "search runbooks"])
    
    observer.emit("LLM_CALL", tokens=1200, cost=0.008, iteration=1)
    observer.emit("REASONING", thought="Service might be degraded, checking health first")
    observer.emit("TOOL_CALL", tool="check_service_health", args={"service_name": "payment-gateway"})
    time.sleep(0.1)
    observer.emit("TOOL_RESULT", tool="check_service_health", status="success", latency_ms=120)
    
    observer.emit("LLM_CALL", tokens=1500, cost=0.010, iteration=2)
    observer.emit("REASONING", thought="Service degraded, checking pods")
    observer.emit("TOOL_CALL", tool="list_pods", args={"namespace": "payments"})
    time.sleep(0.05)
    observer.emit("TOOL_RESULT", tool="list_pods", status="success", latency_ms=85)
    
    observer.emit("LLM_CALL", tokens=1800, cost=0.012, iteration=3)
    observer.emit("TOOL_CALL", tool="get_pod_logs", args={"pod_name": "payment-gw-def34"})
    time.sleep(0.08)
    observer.emit("TOOL_RESULT", tool="get_pod_logs", status="success", latency_ms=95)
    
    observer.emit("LLM_CALL", tokens=1400, cost=0.009, iteration=4)
    observer.emit("REASONING", thought="OOMKilled confirmed, searching runbooks")
    observer.emit("TOOL_CALL", tool="search_runbooks", args={"query": "OOMKilled recovery"})
    observer.emit("TOOL_RESULT", tool="search_runbooks", status="success", latency_ms=200)
    
    observer.emit("LLM_CALL", tokens=2000, cost=0.015, iteration=5)
    observer.emit("DONE", root_cause="OOMKilled from BatchReconciliation")
    
    # Print everything
    print("=" * 70)
    print("INVESTIGATION TRACE")
    print("=" * 70)
    
    for entry in observer.get_timeline():
        print(f"  [{entry['offset_s']:>5.1f}s] {entry['type']:15s} | {json.dumps({k:v for k,v in entry.items() if k not in ('time','offset_s','type')})[:80]}")
    
    print(f"\n{'=' * 70}")
    print("METRICS")
    print(f"{'=' * 70}")
    metrics = observer.get_metrics()
    for k, v in metrics.items():
        if k != "tool_latencies":
            print(f"  {k}: {v}")
    
    print(f"\n  Tool latencies:")
    for tool, stats in metrics.get("tool_latencies", {}).items():
        print(f"    {tool}: avg={stats['avg_ms']}ms, max={stats['max_ms']}ms ({stats['calls']} calls)")
    
    if metrics["anomalies"]:
        print(f"\n  ⚠️  Anomalies detected:")
        for a in metrics["anomalies"]:
            print(f"    - {a}")
    else:
        print(f"\n  ✅ No anomalies detected")
```

---

## 📝 Drills (20 min)

### Drill 1: Memory-Accelerated Investigation

Run 5 investigations with the same agent instance (Exercise 1). Track:
- Does Investigation #2 on the same topic use fewer iterations than #1?
- Does the agent reference past findings in its analysis?
- After 5 investigations, what does the memory contain?

### Drill 2: Graceful Degradation Test

Modify Exercise 2's flaky tools to have a 50% failure rate. Does the agent still produce a useful (partial) analysis? What information gaps does it report?

### Drill 3: Integrate Observability into the Production Agent

Combine Exercise 1 (persistent agent) with Exercise 3 (observability):

```python
# In investigate(), wrap every action:
self.observer.emit("TOOL_CALL", tool=name, args=args)
result = executor.execute_with_retry(name, args, execute_tool)
self.observer.emit("TOOL_RESULT", tool=name, status=result["status"], 
                   latency_ms=latency)
```

At the end, print the full trace and metrics. Store the trace alongside the episode in memory.

### Drill 4: Agent Self-Evaluation

After the agent produces its final answer, add a self-evaluation step:

```python
eval_response = client.messages.create(
    model="claude-haiku-4-5-20241022",  # Cheap model for eval
    system="Rate this SRE investigation analysis on: completeness (0-10), accuracy (0-10), actionability (0-10).",
    messages=[{"role": "user", "content": f"Analysis:\n{answer}"}]
)
```

Track these scores over time. Are investigations getting better as memory accumulates?

---

## ✅ Week 15 Checklist

Before moving to Week 16:

- [ ] Agent has persistent episodic memory (stores/recalls past investigations)
- [ ] Agent recalls similar past cases and references them in analysis
- [ ] Error recovery handles: timeouts, permission errors, exceptions
- [ ] Full observability: structured logs, metrics, traces
- [ ] Anomaly detection: stuck loops, high cost, slow investigations
- [ ] Can replay an investigation from its trace
- [ ] Understand when memory helps (recurring issues) vs when it doesn't (novel issues)

---

## 🧠 Key Concepts from This Week

### Production Agent Stack

```
┌─────────────────────────────────────┐
│  AGENT                               │
│  ├── ReAct Loop (Week 14)           │
│  ├── Planning (Week 14)             │
│  ├── Guardrails (Week 14)           │
│  ├── Persistent Memory (Week 15) ★  │
│  ├── Error Recovery (Week 15) ★     │
│  └── Observability (Week 15) ★      │
│                                       │
│  TOOLS (via MCP)                     │
│  ├── Kubernetes (Week 12)           │
│  ├── ClickHouse Logs (Week 12)      │
│  └── RAG Runbooks (Week 8-10)       │
└─────────────────────────────────────┘
```

### Memory Types for Agents

```
Working Memory:   Conversation history within one investigation (messages list)
Episodic Memory:  Past investigation summaries (vector DB, this week)
Semantic Memory:  Runbooks, documentation (RAG, Weeks 7-10)
Procedural Memory: Investigation plans, prompt templates (MCP prompts, Week 13)
```

### Agent Quality Metrics (Production Dashboard)

```
Metric                    Target         Alert If
──────────────────────    ──────         ────────
Avg iterations            4-6            > 8
Avg cost per investigation $0.02-0.05   > $0.20
Avg duration              10-30s         > 60s
Tool success rate         > 95%          < 80%
Root cause found rate     > 85%          < 70%
Memory recall rate        > 40%          N/A (informational)
```

---

*Next week: Multi-Agent Systems — specialized agents that collaborate. One agent investigates Kubernetes, another queries logs, a coordinator synthesizes findings. Divide and conquer for complex incidents.*
