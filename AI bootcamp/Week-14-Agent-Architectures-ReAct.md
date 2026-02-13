# Week 14: Agent Architectures — The ReAct Pattern

## 🎯 Learning Objectives

By the end of this week, you will:
- Understand what makes an "agent" different from a chatbot or tool-use script
- Master the ReAct pattern (Reason → Act → Observe → repeat)
- Build a single-agent SRE investigator that autonomously gathers data and proposes fixes
- Implement agent memory (scratchpad) so the agent tracks what it's already done
- Add decision-making guardrails (max iterations, cost limits, human-in-the-loop)
- See how everything from Weeks 1-13 combines into one autonomous system

---

## 📖 Theory (20 min)

### What Makes an Agent an "Agent"?

You've already built the pieces. Here's how they compare:

| | Chatbot | Tool-Use Script | Agent |
|---|---|---|---|
| Makes API calls | ✅ | ✅ | ✅ |
| Uses tools | ❌ | ✅ | ✅ |
| Decides WHICH tools | ❌ | Hardcoded | ✅ Autonomously |
| Decides WHEN to stop | N/A | Hardcoded | ✅ Autonomously |
| Adjusts plan mid-task | ❌ | ❌ | ✅ Based on observations |
| Remembers past actions | ❌ | ❌ | ✅ Scratchpad/memory |
| Handles unexpected results | ❌ | Crashes or ignores | ✅ Adapts |

**An agent is an LLM that can reason about what to do, take actions, observe results, and adapt its plan — in a loop — until the task is complete.**

### The ReAct Pattern

ReAct = **Re**asoning + **Act**ing. The agent alternates between thinking and doing:

```
THOUGHT: "The user says payment gateway is down. I should check its health first."
ACTION:  check_service_health("payment-gateway")
OBSERVATION: {status: "degraded", error_rate: 4.7%, pods: "3/5"}

THOUGHT: "Error rate is high and 2 pods are down. I need to see what's wrong with the pods."
ACTION:  list_pods("payments", status_filter="CrashLoopBackOff")
OBSERVATION: {pods: [{name: "payment-gw-def34", restarts: 47}, ...]}

THOUGHT: "47 restarts on def34. Let me check the logs to find the crash reason."
ACTION:  get_pod_logs("payments", "payment-gw-def34")
OBSERVATION: {logs: ["ERROR OutOfMemoryError: Java heap space"]}

THOUGHT: "It's an OOM issue. The pod's memory limit is probably too low. Let me check the pod spec."
ACTION:  describe_pod("payments", "payment-gw-def34")  
OBSERVATION: {resources: {limits: {memory: "512Mi"}}, last_state: "OOMKilled"}

THOUGHT: "Confirmed: OOMKilled with 512Mi limit. A new batch job is loading too much data. 
          I have enough information to provide a complete analysis."
FINAL ANSWER: [comprehensive incident analysis with root cause and fix]
```

Notice: the agent **decided** what to check next based on what it found. A script would have hardcoded the sequence. The agent adapts.

### Agent Architecture

```
┌─────────────────────────────────────────────────┐
│  AGENT                                           │
│                                                   │
│  ┌──────────┐   ┌──────────┐   ┌──────────────┐ │
│  │ REASONER │──→│ PLANNER  │──→│ TOOL EXECUTOR│ │
│  │ (LLM)    │   │ (LLM)    │   │ (code)       │ │
│  └──────────┘   └──────────┘   └──────────────┘ │
│       ↑                              │           │
│       │         ┌──────────┐         │           │
│       └─────────│ MEMORY   │←────────┘           │
│                 │(scratchpad)│                    │
│                 └──────────┘                     │
│                                                   │
│  ┌─────────────────────────────────────────────┐ │
│  │ GUARDRAILS                                   │ │
│  │ • Max iterations (prevent infinite loops)    │ │
│  │ • Cost limit (prevent runaway spending)      │ │
│  │ • Dangerous action detection (no delete!)    │ │
│  │ • Human approval for destructive actions     │ │
│  └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
         ↕               ↕              ↕
    [SRE Tools]    [Log Queries]   [Runbook KB]
```

### Why Agents Matter for SRE

When your pager goes off at 3 AM, the investigation follows a pattern:
1. Check what service is affected
2. Look at the pods/containers
3. Read the logs
4. Check recent deployments
5. Find the runbook
6. Follow the runbook steps

An agent can do steps 1-5 **autonomously** in 30 seconds, presenting the on-call engineer with a complete picture and recommended actions. The human still decides what to do, but the investigation is 10x faster.

---

## 🔨 Hands-On (50 min)

### Exercise 1: The ReAct Agent Loop — From Scratch (25 min)

**Goal:** Build the core ReAct agent loop with explicit reasoning, tool selection, and observation tracking. No frameworks — just Python and the Anthropic API.

Create `week14/react_agent.py`:

```python
"""
Week 14, Exercise 1: ReAct Agent — Built from Scratch

This is the foundational agent pattern:
  THINK → ACT → OBSERVE → THINK → ACT → OBSERVE → ... → ANSWER

No frameworks. Just Python + Claude + your tools.
Understanding this loop is essential before using any agent framework.
"""
import anthropic
import json
import time
from typing import Optional
from dataclasses import dataclass, field

client = anthropic.Anthropic()


# ============================================================
# TOOLS (same as Week 12 — simulated SRE tools)
# ============================================================

SIMULATED_DATA = {
    "services": {
        "payment-gateway": {"status": "degraded", "error_rate": 4.7, "p99_ms": 2100, "pods": "3/5", "alerts": ["ErrorRateHigh", "PodCrashLoopBackOff"]},
        "auth-service": {"status": "healthy", "error_rate": 0.01, "p99_ms": 42, "pods": "4/4", "alerts": []},
        "fraud-detection": {"status": "healthy", "error_rate": 0.05, "p99_ms": 230, "pods": "6/6", "alerts": []},
    },
    "pods": {
        "payments": [
            {"name": "payment-gw-abc12", "status": "Running", "restarts": 0},
            {"name": "payment-gw-def34", "status": "CrashLoopBackOff", "restarts": 47},
            {"name": "payment-gw-ghi56", "status": "CrashLoopBackOff", "restarts": 32},
            {"name": "payment-gw-jkl78", "status": "Running", "restarts": 0},
            {"name": "payment-gw-mno90", "status": "Running", "restarts": 0},
        ]
    },
    "logs": {
        "payment-gw-def34": [
            "14:23:06 WARN  [BatchReconciliation] Loading 2.1M transactions into memory",
            "14:23:08 WARN  [GC] Full GC — heap at 97%",
            "14:23:11 ERROR [BatchReconciliation] java.lang.OutOfMemoryError: Java heap space",
            "14:23:11 ERROR   at BatchReconciliation.loadAllTransactions(BatchReconciliation.java:142)",
        ],
    },
    "pod_details": {
        "payment-gw-def34": {
            "status": "CrashLoopBackOff", "last_state": "OOMKilled (exit 137)",
            "resources": {"limits": {"memory": "512Mi", "cpu": "500m"}},
            "restart_count": 47, "image": "visa/payment-gw:v2.3.1",
        }
    },
    "deployments": {
        "payment-gateway": [
            {"version": "v2.3.1", "time": "14:00 today", "changes": "Added BatchReconciliation job"},
            {"version": "v2.3.0", "time": "yesterday", "changes": "Timeout handling fix"},
        ]
    },
    "runbooks": {
        "oom": "OOMKilled Recovery: 1) kubectl top pods to check memory, 2) kubectl edit deployment to increase resources.limits.memory from 512Mi to 1Gi or 2Gi, 3) kubectl rollout restart deployment, 4) Verify with kubectl rollout status. If batch job is the cause, refactor to use streaming/pagination instead of loading all data into memory.",
    }
}

def execute_tool(name: str, args: dict) -> dict:
    """Execute a tool and return results."""
    if name == "check_service_health":
        svc = args["service_name"]
        return SIMULATED_DATA["services"].get(svc, {"error": f"Unknown: {svc}"})
    
    elif name == "list_pods":
        ns = args["namespace"]
        pods = SIMULATED_DATA["pods"].get(ns, [])
        if args.get("status_filter"):
            pods = [p for p in pods if p["status"] == args["status_filter"]]
        return {"namespace": ns, "pods": pods}
    
    elif name == "get_pod_logs":
        pod = args["pod_name"]
        return {"pod": pod, "logs": SIMULATED_DATA["logs"].get(pod, ["No logs found"])}
    
    elif name == "describe_pod":
        pod = args["pod_name"]
        return SIMULATED_DATA["pod_details"].get(pod, {"error": "Pod not found"})
    
    elif name == "get_recent_deploys":
        svc = args["service_name"]
        return {"deployments": SIMULATED_DATA["deployments"].get(svc, [])}
    
    elif name == "search_runbooks":
        query = args["query"].lower()
        results = []
        for key, content in SIMULATED_DATA["runbooks"].items():
            if any(word in query for word in key.split()):
                results.append({"title": key, "content": content})
        return {"results": results if results else [{"title": "No match", "content": "No runbook found for this query."}]}
    
    return {"error": f"Unknown tool: {name}"}


# Tool definitions for the API
TOOLS = [
    {"name": "check_service_health", "description": "Check service health: status, error rate, latency, alerts. Use FIRST when investigating.",
     "input_schema": {"type": "object", "properties": {"service_name": {"type": "string", "enum": ["payment-gateway", "auth-service", "fraud-detection"]}}, "required": ["service_name"]}},
    {"name": "list_pods", "description": "List Kubernetes pods with status and restart counts.",
     "input_schema": {"type": "object", "properties": {"namespace": {"type": "string"}, "status_filter": {"type": "string"}}, "required": ["namespace"]}},
    {"name": "get_pod_logs", "description": "Get pod log lines. Use for error messages and stack traces.",
     "input_schema": {"type": "object", "properties": {"pod_name": {"type": "string"}, "lines": {"type": "integer", "default": 20}}, "required": ["pod_name"]}},
    {"name": "describe_pod", "description": "Detailed pod info: resources, limits, last termination reason.",
     "input_schema": {"type": "object", "properties": {"pod_name": {"type": "string"}}, "required": ["pod_name"]}},
    {"name": "get_recent_deploys", "description": "Deployment history for a service.",
     "input_schema": {"type": "object", "properties": {"service_name": {"type": "string"}}, "required": ["service_name"]}},
    {"name": "search_runbooks", "description": "Search SRE runbooks for troubleshooting steps.",
     "input_schema": {"type": "object", "properties": {"query": {"type": "string"}}, "required": ["query"]}},
]


# ============================================================
# AGENT MEMORY / SCRATCHPAD
# ============================================================

@dataclass
class AgentMemory:
    """Tracks what the agent has done and discovered."""
    task: str
    steps: list = field(default_factory=list)
    findings: list = field(default_factory=list)
    tools_called: list = field(default_factory=list)
    total_tokens: int = 0
    total_cost: float = 0.0
    start_time: float = field(default_factory=time.time)
    
    def add_step(self, tool_name: str, arguments: dict, result_summary: str):
        self.steps.append({
            "step": len(self.steps) + 1,
            "tool": tool_name,
            "args": arguments,
            "result_summary": result_summary[:200]
        })
        self.tools_called.append(tool_name)
    
    def add_finding(self, finding: str):
        self.findings.append(finding)
    
    def get_summary(self) -> str:
        elapsed = time.time() - self.start_time
        return (
            f"Task: {self.task}\n"
            f"Steps taken: {len(self.steps)}\n"
            f"Tools called: {self.tools_called}\n"
            f"Total tokens: {self.total_tokens}\n"
            f"Estimated cost: ${self.total_cost:.4f}\n"
            f"Time elapsed: {elapsed:.1f}s"
        )


# ============================================================
# THE REACT AGENT
# ============================================================

class SREAgent:
    """
    A ReAct-pattern agent for SRE investigation.
    
    THINK → ACT → OBSERVE → repeat until solved.
    """
    
    def __init__(
        self,
        max_iterations: int = 10,
        max_cost_usd: float = 0.50,
        model: str = "claude-sonnet-4-5-20250514"
    ):
        self.max_iterations = max_iterations
        self.max_cost = max_cost_usd
        self.model = model
        self.client = anthropic.Anthropic()
    
    def investigate(self, task: str) -> dict:
        """
        Run an autonomous investigation.
        
        The agent will:
        1. Think about what to do
        2. Call tools to gather data
        3. Observe results
        4. Repeat until it has enough info
        5. Deliver a final analysis
        """
        memory = AgentMemory(task=task)
        
        system_prompt = """You are an expert SRE agent investigating production issues autonomously.

YOUR PROCESS:
1. Start by checking service health to understand the scope
2. Drill into unhealthy components (pods, logs, deployments)
3. Search runbooks for relevant procedures
4. Only stop when you have enough data for a root cause analysis

INVESTIGATION RULES:
- Always start with check_service_health
- If pods are unhealthy, get their logs AND describe them
- Always check recent deployments when something is broken
- Search runbooks for the specific issue you found
- Don't repeat the same tool call with the same arguments
- Stop after finding root cause — don't keep investigating if you know the answer

FINAL ANSWER FORMAT:
When you have enough information, provide your analysis as text (not a tool call):

SEVERITY: P1/P2/P3/P4
ROOT CAUSE: [one paragraph explaining what went wrong and why]
EVIDENCE: [list the key data points that support your conclusion]
IMMEDIATE ACTIONS: [numbered list of specific commands to run NOW]
LONG-TERM FIX: [what to change to prevent recurrence]
RUNBOOK: [reference to relevant runbook if found]"""

        messages = [{"role": "user", "content": task}]
        
        print(f"\n{'🤖' * 35}")
        print(f"AGENT INVESTIGATION: {task}")
        print(f"{'🤖' * 35}\n")
        
        for iteration in range(self.max_iterations):
            # ---- GUARDRAIL: Cost check ----
            if memory.total_cost >= self.max_cost:
                print(f"\n⚠️  Cost limit reached (${memory.total_cost:.4f} >= ${self.max_cost})")
                break
            
            # ---- THINK + ACT: Ask Claude what to do ----
            response = self.client.messages.create(
                model=self.model,
                max_tokens=2000,
                temperature=0,
                system=system_prompt,
                tools=TOOLS,
                messages=messages
            )
            
            # Track costs
            input_tokens = response.usage.input_tokens
            output_tokens = response.usage.output_tokens
            memory.total_tokens += input_tokens + output_tokens
            memory.total_cost += (input_tokens * 3 + output_tokens * 15) / 1_000_000
            
            # ---- CHECK: Is the agent done? ----
            if response.stop_reason == "end_turn":
                # Agent is done — extract final answer
                final_text = ""
                for block in response.content:
                    if block.type == "text":
                        final_text += block.text
                
                print(f"\n{'=' * 70}")
                print(f"📋 FINAL ANALYSIS (after {iteration + 1} iterations)")
                print(f"{'=' * 70}\n")
                print(final_text)
                
                return {
                    "answer": final_text,
                    "memory": memory,
                    "iterations": iteration + 1,
                    "summary": memory.get_summary()
                }
            
            # ---- ACT: Execute tool calls ----
            if response.stop_reason == "tool_use":
                tool_results = []
                
                for block in response.content:
                    if block.type == "text" and block.text.strip():
                        # Agent's reasoning (thinking out loud)
                        print(f"💭 [{iteration+1}] Thinking: {block.text.strip()[:120]}")
                    
                    if block.type == "tool_use":
                        tool_name = block.name
                        tool_args = block.input
                        
                        # ---- GUARDRAIL: Duplicate check ----
                        call_sig = f"{tool_name}({json.dumps(tool_args, sort_keys=True)})"
                        if call_sig in [f"{s['tool']}({json.dumps(s['args'], sort_keys=True)})" for s in memory.steps]:
                            print(f"⚠️  [{iteration+1}] Skipping duplicate: {tool_name}")
                            tool_results.append({
                                "type": "tool_result",
                                "tool_use_id": block.id,
                                "content": json.dumps({"note": "Already called this with same arguments. Use previous result."})
                            })
                            continue
                        
                        # Execute the tool
                        print(f"🔧 [{iteration+1}] Action: {tool_name}({json.dumps(tool_args)})")
                        
                        result = execute_tool(tool_name, tool_args)
                        result_str = json.dumps(result)
                        
                        # ---- OBSERVE: Show what the agent found ----
                        preview = result_str[:150] + "..." if len(result_str) > 150 else result_str
                        print(f"   👁  Observed: {preview}")
                        
                        # Update memory
                        memory.add_step(tool_name, tool_args, result_str)
                        
                        tool_results.append({
                            "type": "tool_result",
                            "tool_use_id": block.id,
                            "content": result_str
                        })
                
                # Add to conversation history
                messages.append({"role": "assistant", "content": response.content})
                messages.append({"role": "user", "content": tool_results})
                print()  # Blank line between iterations
        
        # Max iterations reached
        print(f"\n⚠️  Max iterations ({self.max_iterations}) reached")
        return {
            "answer": "Investigation incomplete — max iterations reached",
            "memory": memory,
            "iterations": self.max_iterations,
            "summary": memory.get_summary()
        }


# ============================================================
# RUN THE AGENT
# ============================================================

if __name__ == "__main__":
    agent = SREAgent(max_iterations=10, max_cost_usd=0.50)
    
    # Investigation 1: Service degradation
    result = agent.investigate(
        "The payment gateway seems degraded. Customers are complaining about failed payments. "
        "Please investigate and tell me the root cause and how to fix it."
    )
    
    print(f"\n{'=' * 70}")
    print("AGENT STATISTICS")
    print(f"{'=' * 70}")
    print(result["summary"])
```

---

### Exercise 2: Agent with Memory and Planning (15 min)

**Goal:** Add explicit planning and a memory scratchpad so the agent tracks its investigation state.

Create `week14/planning_agent.py`:

```python
"""
Week 14, Exercise 2: Agent with Explicit Planning

Adds two capabilities:
1. PLANNING: Before investigating, the agent creates a plan
2. MEMORY: Agent maintains a scratchpad of findings between steps

This produces more structured, thorough investigations.
"""
import anthropic
import json

client = anthropic.Anthropic()

# (Reuse TOOLS and execute_tool from Exercise 1)
from react_agent import TOOLS, execute_tool


def create_investigation_plan(task: str) -> list[str]:
    """
    Step 1: Ask Claude to create an investigation plan BEFORE acting.
    This produces better results than jumping straight into tool calls.
    """
    response = client.messages.create(
        model="claude-sonnet-4-5-20250514",
        max_tokens=500,
        temperature=0,
        system="You are an SRE planning an investigation. Create a numbered plan of what tools to use and in what order. Be specific but concise. 5-8 steps max.",
        messages=[{
            "role": "user",
            "content": f"""Create an investigation plan for:

{task}

Available tools: check_service_health, list_pods, get_pod_logs, describe_pod, get_recent_deploys, search_runbooks

Return ONLY a numbered list of steps. Each step should name the specific tool and what to look for."""
        }]
    )
    
    plan_text = response.content[0].text
    # Parse numbered steps
    steps = [line.strip() for line in plan_text.split('\n') if line.strip() and line.strip()[0].isdigit()]
    return steps


def investigate_with_plan(task: str, max_iterations: int = 10) -> str:
    """
    Agent that plans first, then executes with a memory scratchpad.
    """
    # ---- PHASE 1: PLAN ----
    print("📋 PHASE 1: Creating investigation plan...\n")
    plan = create_investigation_plan(task)
    for step in plan:
        print(f"  {step}")
    
    # ---- PHASE 2: EXECUTE with scratchpad ----
    print(f"\n🔍 PHASE 2: Executing plan...\n")
    
    scratchpad = []  # Accumulates findings
    
    system = f"""You are an SRE agent executing an investigation plan.

INVESTIGATION PLAN:
{chr(10).join(plan)}

RULES:
- Follow the plan steps in order
- After each tool result, update your scratchpad with key findings
- Adapt the plan if you discover something unexpected
- When you have enough data, provide your final analysis

SCRATCHPAD (findings so far):
{{scratchpad}}

After gathering ALL needed data, provide your FINAL ANALYSIS."""

    messages = [{"role": "user", "content": task}]
    
    for iteration in range(max_iterations):
        # Inject current scratchpad into system prompt
        current_system = system.replace("{scratchpad}", 
            "\n".join(f"- {f}" for f in scratchpad) if scratchpad else "(empty)")
        
        response = client.messages.create(
            model="claude-sonnet-4-5-20250514",
            max_tokens=2000,
            temperature=0,
            system=current_system,
            tools=TOOLS,
            messages=messages
        )
        
        if response.stop_reason == "end_turn":
            final = ""
            for block in response.content:
                if block.type == "text":
                    final += block.text
            
            print(f"\n{'=' * 70}")
            print(f"📋 FINAL ANALYSIS")
            print(f"{'=' * 70}\n")
            print(final)
            
            print(f"\n📝 Scratchpad at end of investigation:")
            for f in scratchpad:
                print(f"  • {f}")
            
            return final
        
        if response.stop_reason == "tool_use":
            tool_results = []
            
            for block in response.content:
                if block.type == "text" and block.text.strip():
                    thought = block.text.strip()
                    print(f"💭 [{iteration+1}] {thought[:120]}")
                    
                    # Extract findings from thoughts and add to scratchpad
                    if any(word in thought.lower() for word in ["found", "shows", "confirms", "indicates", "appears", "discovered"]):
                        scratchpad.append(thought[:100])
                
                if block.type == "tool_use":
                    print(f"🔧 [{iteration+1}] {block.name}({json.dumps(block.input)})")
                    
                    result = execute_tool(block.name, block.input)
                    result_str = json.dumps(result)
                    
                    # Auto-extract key findings from results
                    result_data = result
                    if isinstance(result_data, dict):
                        if result_data.get("status") == "degraded":
                            scratchpad.append(f"Service {block.input.get('service_name', '?')} is DEGRADED")
                        if "OOMKilled" in result_str:
                            scratchpad.append("OOMKilled detected — memory exhaustion")
                        if "CrashLoopBackOff" in result_str:
                            scratchpad.append("Pods in CrashLoopBackOff")
                        if "OutOfMemoryError" in result_str:
                            scratchpad.append("Java OutOfMemoryError in application")
                    
                    preview = result_str[:120] + "..." if len(result_str) > 120 else result_str
                    print(f"   👁  {preview}")
                    
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result_str
                    })
            
            messages.append({"role": "assistant", "content": response.content})
            messages.append({"role": "user", "content": tool_results})
            print()
    
    return "Investigation incomplete"


if __name__ == "__main__":
    investigate_with_plan(
        "The payment gateway is degraded. Error rate is high and customers "
        "can't complete payments. Investigate the root cause and recommend a fix."
    )
```

---

### Exercise 3: Agent Guardrails & Human-in-the-Loop (10 min)

**Goal:** Add safety guardrails — cost limits, dangerous action detection, and human confirmation for risky operations.

Create `week14/agent_guardrails.py`:

```python
"""
Week 14, Exercise 3: Agent Guardrails

Production agents need safety rails:
1. Max iterations (prevent infinite loops)
2. Cost cap (prevent runaway spending)  
3. Dangerous action detection (catch destructive commands)
4. Human-in-the-loop (ask for approval before risky actions)
5. Timeout (don't investigate forever)
"""
import time
from dataclasses import dataclass


@dataclass
class GuardrailConfig:
    """Configuration for agent safety guardrails."""
    max_iterations: int = 10
    max_cost_usd: float = 0.50
    max_time_seconds: int = 120
    require_approval_for: list = None  # Tool names that need human approval
    blocked_tools: list = None          # Tools that are never allowed
    
    def __post_init__(self):
        if self.require_approval_for is None:
            self.require_approval_for = [
                "scale_deployment",
                "rollback_deployment",
                "restart_pods",
                "kill_process",
                "flush_cache",
                "terminate_connection",
            ]
        if self.blocked_tools is None:
            self.blocked_tools = [
                "delete_deployment",
                "delete_namespace",
                "drop_database",
                "delete_pvc",
            ]


class AgentGuardrails:
    """
    Safety guardrails for SRE agents.
    
    Check these BEFORE every tool execution.
    """
    
    def __init__(self, config: GuardrailConfig = None):
        self.config = config or GuardrailConfig()
        self.iteration_count = 0
        self.total_cost = 0.0
        self.start_time = time.time()
        self.action_log = []
    
    def check_iteration(self) -> tuple[bool, str]:
        """Check if we've exceeded max iterations."""
        self.iteration_count += 1
        if self.iteration_count > self.config.max_iterations:
            return False, f"Max iterations ({self.config.max_iterations}) exceeded"
        return True, ""
    
    def check_cost(self, additional_cost: float) -> tuple[bool, str]:
        """Check if adding this cost would exceed the budget."""
        projected = self.total_cost + additional_cost
        if projected > self.config.max_cost_usd:
            return False, f"Cost limit: ${projected:.4f} would exceed ${self.config.max_cost_usd} budget"
        self.total_cost += additional_cost
        return True, ""
    
    def check_time(self) -> tuple[bool, str]:
        """Check if we've exceeded the time limit."""
        elapsed = time.time() - self.start_time
        if elapsed > self.config.max_time_seconds:
            return False, f"Time limit: {elapsed:.0f}s exceeds {self.config.max_time_seconds}s"
        return True, ""
    
    def check_tool(self, tool_name: str, arguments: dict) -> tuple[bool, str]:
        """
        Check if a tool call is allowed.
        Returns (allowed, reason).
        """
        # Blocked tools — never allow
        if tool_name in self.config.blocked_tools:
            return False, f"BLOCKED: '{tool_name}' is a destructive operation and is not allowed"
        
        # Dangerous content in arguments
        dangerous_patterns = ["rm -rf", "DROP TABLE", "DELETE FROM", "TRUNCATE", "FORMAT"]
        args_str = str(arguments).upper()
        for pattern in dangerous_patterns:
            if pattern in args_str:
                return False, f"BLOCKED: Dangerous pattern '{pattern}' detected in arguments"
        
        # Require human approval
        if tool_name in self.config.require_approval_for:
            return self._request_approval(tool_name, arguments)
        
        # Log the action
        self.action_log.append({
            "tool": tool_name,
            "args": arguments,
            "time": time.time() - self.start_time
        })
        
        return True, ""
    
    def _request_approval(self, tool_name: str, arguments: dict) -> tuple[bool, str]:
        """Ask the human for approval before executing a risky action."""
        print(f"\n{'⚠️' * 30}")
        print(f"HUMAN APPROVAL REQUIRED")
        print(f"{'⚠️' * 30}")
        print(f"\n  Tool: {tool_name}")
        print(f"  Arguments: {arguments}")
        print(f"\n  This is a potentially risky operation.")
        
        # In production, this could be a Slack message, PagerDuty prompt, etc.
        # For now, use terminal input
        response = input("\n  Approve? (yes/no): ").strip().lower()
        
        if response in ("yes", "y"):
            self.action_log.append({
                "tool": tool_name,
                "args": arguments,
                "approved_by": "human",
                "time": time.time() - self.start_time
            })
            return True, "Approved by human"
        else:
            return False, "Rejected by human operator"
    
    def check_all(self, tool_name: str, arguments: dict, additional_cost: float = 0) -> tuple[bool, str]:
        """Run ALL guardrail checks. Call this before every tool execution."""
        
        ok, msg = self.check_iteration()
        if not ok: return False, msg
        
        ok, msg = self.check_time()
        if not ok: return False, msg
        
        ok, msg = self.check_cost(additional_cost)
        if not ok: return False, msg
        
        ok, msg = self.check_tool(tool_name, arguments)
        if not ok: return False, msg
        
        return True, "All checks passed"
    
    def get_report(self) -> dict:
        """Get a summary of guardrail activity."""
        return {
            "iterations": self.iteration_count,
            "total_cost": round(self.total_cost, 4),
            "elapsed_seconds": round(time.time() - self.start_time, 1),
            "actions_taken": len(self.action_log),
            "actions": self.action_log
        }


# ============================================================
# TEST GUARDRAILS
# ============================================================

if __name__ == "__main__":
    guardrails = AgentGuardrails()
    
    # Safe tool — should pass
    ok, msg = guardrails.check_all("check_service_health", {"service_name": "payment-gateway"})
    print(f"check_service_health: {'✅ PASS' if ok else '❌ FAIL'} — {msg}")
    
    # Blocked tool — should fail
    ok, msg = guardrails.check_all("delete_namespace", {"namespace": "payments"})
    print(f"delete_namespace: {'✅ PASS' if ok else '❌ FAIL'} — {msg}")
    
    # Dangerous argument — should fail
    ok, msg = guardrails.check_all("search_logs", {"keyword": "'; DROP TABLE logs; --"})
    print(f"SQL injection: {'✅ PASS' if ok else '❌ FAIL'} — {msg}")
    
    # Approval-required tool — will prompt
    # (Uncomment to test interactively)
    # ok, msg = guardrails.check_all("rollback_deployment", {"deployment": "payment-gw"})
    
    print(f"\n📊 Guardrail report: {guardrails.get_report()}")
```

---

## 📝 Drills (20 min)

### Drill 1: Integrate Guardrails into the ReAct Agent

Modify `react_agent.py` to use the `AgentGuardrails` class:

```python
# In the investigation loop, BEFORE every tool execution:
ok, msg = guardrails.check_all(tool_name, tool_args, estimated_cost)
if not ok:
    # Return error to Claude instead of executing the tool
    tool_results.append({
        "type": "tool_result",
        "tool_use_id": block.id,
        "content": json.dumps({"error": f"Guardrail blocked: {msg}"}),
        "is_error": True
    })
    continue
```

Test that the agent stops when it hits the cost limit or tries a blocked action.

### Drill 2: Different Investigation Scenarios

Run your agent on these different scenarios and compare behavior:

1. "The auth-service is returning 401 errors" (should find healthy service)
2. "Disk space is filling up on the monitoring nodes" (should use different tools)
3. "Multiple services seem slow today" (should check multiple services)

Does the agent adapt its investigation strategy to each scenario?

### Drill 3: Agent Comparison — With vs Without Planning

Run the same investigation twice:
- Without planning (Exercise 1 agent)
- With planning (Exercise 2 agent)

Compare: number of tool calls, relevance of tools chosen, quality of final analysis. Does planning produce better results?

### Drill 4: Agent Observability Dashboard

Create a function that produces a structured report of the agent's investigation:

```python
def generate_investigation_report(memory: AgentMemory) -> dict:
    return {
        "timeline": [
            {"step": s["step"], "tool": s["tool"], "time_offset": "...", "finding": s["result_summary"][:50]}
            for s in memory.steps
        ],
        "tools_used": list(set(memory.tools_called)),
        "unique_tools": len(set(memory.tools_called)),
        "total_calls": len(memory.tools_called),
        "efficiency": len(set(memory.tools_called)) / max(len(memory.tools_called), 1),
        "cost": memory.total_cost,
        "findings": memory.findings,
    }
```

An efficiency score near 1.0 means the agent never repeated tools. Near 0.5 means lots of redundant calls.

---

## ✅ Week 14 Checklist

Before moving to Week 15:

- [ ] Can explain what makes an agent different from a tool-use script
- [ ] Understand the ReAct pattern (Reason → Act → Observe → loop)
- [ ] Have a working SRE investigation agent that calls tools autonomously
- [ ] Agent has memory/scratchpad that tracks findings
- [ ] Agent can create an investigation plan before executing
- [ ] Guardrails prevent: infinite loops, cost overruns, dangerous actions
- [ ] Human-in-the-loop pattern for risky operations

---

## 🧠 Key Concepts from This Week

### The Agent Equation

```
Agent = LLM + Tools + Loop + Memory + Guardrails

LLM:        Reasoning engine (Claude)
Tools:      Actions the agent can take (Weeks 6, 12-13)
Loop:       ReAct cycle (this week)
Memory:     Scratchpad of findings (this week)
Guardrails: Safety limits (this week)
```

### ReAct Pattern (memorize this)

```python
while not done:
    response = llm(messages, tools)      # THINK + decide action
    
    if response.stop_reason == "end_turn":
        return response.text              # DONE — deliver answer
    
    for tool_call in response.tool_calls:
        if guardrails.check(tool_call):   # SAFETY CHECK
            result = execute(tool_call)    # ACT
            memory.record(tool_call, result) # REMEMBER
            messages.append(result)        # OBSERVE (feed back)
```

### Agent Quality Metrics

```
Metric              Good Agent    Bad Agent
─────────────────   ──────────    ─────────
Iterations          3-6           15+ (wandering)
Unique tools used   4-6           2 (not investigating enough)
Duplicate calls     0             5+ (repeating itself)
Time to answer      10-30s        60s+ (too slow)
Cost per task       $0.02-0.10    $0.50+ (too expensive)
Root cause found    Yes           Vague/wrong
```

---

*Next week: Production Agents — multi-tool with persistent memory, error recovery, agent observability, and the `sre-agent` portfolio project.*
