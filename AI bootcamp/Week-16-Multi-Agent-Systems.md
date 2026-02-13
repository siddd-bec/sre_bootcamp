# Week 16: Multi-Agent Systems

## 🎯 Learning Objectives

By the end of this week, you will:
- Understand when single agents struggle and multi-agent systems shine
- Implement the Coordinator pattern — a manager agent that delegates to specialists
- Build specialized agents (K8s expert, Log analyst, Runbook advisor) that collaborate
- Handle inter-agent communication and result synthesis
- Implement parallel agent execution for faster investigations
- Know the tradeoffs: single-agent simplicity vs multi-agent power

---

## 📖 Theory (20 min)

### When Single Agents Break Down

Your Week 14-15 agent works great for straightforward investigations. But complex incidents overwhelm a single agent:

**Problem 1: Context window saturation.** A P1 incident touching 5 services generates massive amounts of data. One agent's context fills up and it loses track of early findings.

**Problem 2: Tool expertise dilution.** Describing a Kubernetes pod, querying ClickHouse logs, and interpreting Redis metrics all require different "mental models." One agent trying to be expert in everything produces shallower analysis.

**Problem 3: Serial bottleneck.** A single agent checks services one by one. Checking 4 services in parallel is 4x faster.

### Multi-Agent Architectures

**Pattern 1: Coordinator + Specialists (this week)**
```
                    ┌─────────────┐
                    │ COORDINATOR │
                    │ (manager)   │
                    └──────┬──────┘
              ┌────────────┼────────────┐
              ↓            ↓            ↓
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ K8s      │ │ Log      │ │ Runbook  │
        │ Expert   │ │ Analyst  │ │ Advisor  │
        └──────────┘ └──────────┘ └──────────┘
```
The Coordinator breaks the task into subtasks, assigns them to specialists, collects results, and synthesizes a final answer.

**Pattern 2: Pipeline (sequential handoff)**
```
Triage Agent → Investigation Agent → Remediation Agent → Verification Agent
```
Each agent does one phase and passes context to the next.

**Pattern 3: Debate / Consensus**
```
Agent A (optimist): "It's just a traffic spike, will resolve itself"
Agent B (pessimist): "The OOM pattern suggests a memory leak"
Judge Agent: weighs both arguments, decides
```

This week focuses on **Pattern 1** — it's the most practical for SRE and the foundation for the Week 20 capstone.

### Specialist Agent Design

Each specialist agent should:
- Have a **narrow, well-defined scope** (one domain)
- Have its **own system prompt** tuned for its specialty
- Access **only the tools it needs** (principle of least privilege)
- Return a **structured report** the coordinator can consume

```
K8s Expert:
  Scope: Pod status, container health, resource usage, deployments
  Tools: list_pods, describe_pod, get_events, list_deployments
  Output: { healthy_pods, unhealthy_pods, resource_issues, deployment_changes }

Log Analyst:
  Scope: Error patterns, log frequency, stack traces, correlations
  Tools: search_logs, count_errors, trace_transaction
  Output: { error_summary, top_errors, patterns, affected_traces }

Runbook Advisor:
  Scope: Find and summarize relevant procedures
  Tools: search_runbooks
  Output: { relevant_runbooks, recommended_actions, escalation_advice }
```

### Inter-Agent Communication

Agents communicate through **structured reports**, not raw conversation. Each specialist produces a JSON report, and the coordinator reads all reports to synthesize.

```python
# Specialist returns structured data
k8s_report = {
    "agent": "k8s-expert",
    "status": "completed",
    "findings": [...],
    "severity_assessment": "P2",
    "confidence": 0.85
}

# Coordinator consumes all reports
coordinator_prompt = f"""
K8s Expert Report: {k8s_report}
Log Analyst Report: {log_report}
Runbook Advisor Report: {runbook_report}

Synthesize these into a single incident analysis.
"""
```

---

## 🔨 Hands-On (50 min)

### Exercise 1: Specialist Agents (20 min)

**Goal:** Build three specialist agents, each with a narrow focus and dedicated tools.

Create `week16/specialist_agents.py`:

```python
"""
Week 16, Exercise 1: Specialist Agents

Three experts, each mastering one domain:
1. K8sExpert — Kubernetes cluster analysis
2. LogAnalyst — Log pattern analysis
3. RunbookAdvisor — Knowledge base search and recommendations
"""
import anthropic
import json

client = anthropic.Anthropic()


# ============================================================
# SIMULATED TOOLS (condensed from Week 12-13)
# ============================================================

def sim_tool(name: str, args: dict) -> dict:
    """Simulated SRE tools."""
    data = {
        ("check_service_health", "payment-gateway"): {
            "status": "degraded", "error_rate": 4.7, "p99_ms": 2100,
            "pods": "3/5", "alerts": ["ErrorRateHigh", "PodCrashLoopBackOff"]},
        ("check_service_health", "auth-service"): {
            "status": "healthy", "error_rate": 0.01, "p99_ms": 42, "pods": "4/4", "alerts": []},
        ("check_service_health", "fraud-detection"): {
            "status": "healthy", "error_rate": 0.05, "p99_ms": 230, "pods": "6/6", "alerts": []},
        ("list_pods", "payments"): {"pods": [
            {"name": "payment-gw-abc12", "status": "Running", "restarts": 0, "cpu": "250m", "memory": "384Mi"},
            {"name": "payment-gw-def34", "status": "CrashLoopBackOff", "restarts": 47, "cpu": "0m", "memory": "512Mi"},
            {"name": "payment-gw-ghi56", "status": "CrashLoopBackOff", "restarts": 32, "cpu": "0m", "memory": "510Mi"},
            {"name": "payment-gw-jkl78", "status": "Running", "restarts": 0, "cpu": "230m", "memory": "360Mi"},
            {"name": "payment-gw-mno90", "status": "Running", "restarts": 0, "cpu": "280m", "memory": "390Mi"},
        ]},
        ("describe_pod", "payment-gw-def34"): {
            "status": "CrashLoopBackOff", "last_state": "OOMKilled (exit 137)",
            "resources": {"limits": {"memory": "512Mi", "cpu": "500m"}},
            "restart_count": 47, "image": "visa/payment-gw:v2.3.1"},
        ("get_events", "payments"): {"events": [
            {"type": "Warning", "reason": "OOMKilled", "object": "pod/payment-gw-def34", "message": "Container exceeded 512Mi"},
            {"type": "Warning", "reason": "BackOff", "object": "pod/payment-gw-def34", "message": "Back-off restarting"},
            {"type": "Normal", "reason": "ScalingReplicaSet", "object": "deployment/payment-gateway", "message": "Scaled to 5"},
        ]},
        ("list_deployments", "payments"): {"deployments": [
            {"name": "payment-gateway", "ready": "3/5", "image": "visa/payment-gw:v2.3.1"}]},
        ("get_recent_deploys", "payment-gateway"): {"deploys": [
            {"version": "v2.3.1", "time": "today 14:00", "changes": "Added BatchReconciliation job"},
            {"version": "v2.3.0", "time": "yesterday", "changes": "Timeout fix"}]},
    }
    
    # Search logs
    if name == "search_logs":
        svc = args.get("service", "")
        level = args.get("level", "")
        logs = [
            {"ts": "14:23:01", "svc": "payment-gateway", "level": "ERROR", "msg": "OutOfMemoryError: Java heap space"},
            {"ts": "14:23:02", "svc": "payment-gateway", "level": "ERROR", "msg": "Transaction tx-def456 timeout 30s"},
            {"ts": "14:23:03", "svc": "payment-gateway", "level": "ERROR", "msg": "Circuit breaker OPEN card-network-gw"},
            {"ts": "14:22:30", "svc": "payment-gateway", "level": "WARN", "msg": "Loading 2.1M transactions into memory"},
            {"ts": "14:22:45", "svc": "payment-gateway", "level": "WARN", "msg": "Full GC — heap at 97%"},
            {"ts": "14:23:01", "svc": "fraud-detection", "level": "ERROR", "msg": "Redis cache miss velocity data"},
        ]
        if svc: logs = [l for l in logs if l["svc"] == svc]
        if level: logs = [l for l in logs if l["level"] == level]
        return {"logs": logs, "count": len(logs)}
    
    if name == "count_errors":
        return {"groups": [
            {"service": "payment-gateway", "count": 3},
            {"service": "fraud-detection", "count": 1}]}
    
    if name == "get_pod_logs":
        return {"logs": [
            "WARN  Loading 2.1M transactions into memory",
            "WARN  Full GC — heap at 97%",
            "ERROR java.lang.OutOfMemoryError: Java heap space",
            "ERROR   at BatchReconciliation.loadAllTransactions:142"]}
    
    if name == "search_runbooks":
        return {"results": [{"title": "OOMKilled Recovery",
            "content": "Increase memory limits (kubectl edit deployment). For batch jobs: refactor to streaming/pagination. Immediate: rollback or increase to 2Gi."}]}
    
    # Generic lookup
    key1 = (name, args.get("service_name") or args.get("namespace") or args.get("pod_name", ""))
    if key1 in data:
        return data[key1]
    
    return {"result": f"{name} completed", "note": "No specific data available"}


# ============================================================
# SPECIALIST AGENT CLASS
# ============================================================

class SpecialistAgent:
    """
    A specialist agent focused on one domain.
    Has its own system prompt, tools, and output format.
    """
    
    def __init__(self, name: str, system_prompt: str, tools: list[dict],
                 model: str = "claude-sonnet-4-5-20250514", max_iterations: int = 5):
        self.name = name
        self.system_prompt = system_prompt
        self.tools = tools
        self.model = model
        self.max_iterations = max_iterations
    
    def investigate(self, task: str) -> dict:
        """
        Run investigation within this specialist's domain.
        Returns a structured report.
        """
        messages = [{"role": "user", "content": task}]
        tool_calls = []
        
        for iteration in range(self.max_iterations):
            response = client.messages.create(
                model=self.model,
                max_tokens=1500,
                temperature=0,
                system=self.system_prompt,
                tools=self.tools,
                messages=messages
            )
            
            if response.stop_reason == "end_turn":
                report_text = "".join(b.text for b in response.content if b.type == "text")
                
                return {
                    "agent": self.name,
                    "status": "completed",
                    "iterations": iteration + 1,
                    "tool_calls": tool_calls,
                    "report": report_text,
                    "tokens": response.usage.input_tokens + response.usage.output_tokens,
                }
            
            if response.stop_reason == "tool_use":
                tool_results = []
                for block in response.content:
                    if block.type == "tool_use":
                        result = sim_tool(block.name, block.input)
                        tool_calls.append({"tool": block.name, "args": block.input})
                        tool_results.append({
                            "type": "tool_result",
                            "tool_use_id": block.id,
                            "content": json.dumps(result)
                        })
                
                messages.append({"role": "assistant", "content": response.content})
                messages.append({"role": "user", "content": tool_results})
        
        return {"agent": self.name, "status": "incomplete", "iterations": self.max_iterations,
                "tool_calls": tool_calls, "report": "Max iterations reached"}


# ============================================================
# DEFINE THREE SPECIALISTS
# ============================================================

k8s_expert = SpecialistAgent(
    name="k8s-expert",
    system_prompt="""You are a Kubernetes expert investigating pod and deployment health.

YOUR FOCUS: Pods, containers, resources, deployments, events.
NOT YOUR FOCUS: Application logs, business logic, runbooks.

INVESTIGATION STEPS:
1. Check service health for overview
2. List pods to find unhealthy ones
3. Describe unhealthy pods for root cause details
4. Check events for warnings
5. Check deployment history

OUTPUT: Provide a structured Kubernetes assessment:
- Which pods are unhealthy and why
- Resource issues (memory limits, CPU)
- Recent deployment changes
- Severity assessment (P1-P4)""",
    tools=[
        {"name": "check_service_health", "description": "Service health overview.",
         "input_schema": {"type": "object", "properties": {"service_name": {"type": "string"}}, "required": ["service_name"]}},
        {"name": "list_pods", "description": "List pods with status.",
         "input_schema": {"type": "object", "properties": {"namespace": {"type": "string"}}, "required": ["namespace"]}},
        {"name": "describe_pod", "description": "Detailed pod info.",
         "input_schema": {"type": "object", "properties": {"pod_name": {"type": "string"}}, "required": ["pod_name"]}},
        {"name": "get_events", "description": "Recent K8s events.",
         "input_schema": {"type": "object", "properties": {"namespace": {"type": "string"}}, "required": ["namespace"]}},
        {"name": "list_deployments", "description": "Deployments with replicas.",
         "input_schema": {"type": "object", "properties": {"namespace": {"type": "string"}}, "required": ["namespace"]}},
        {"name": "get_recent_deploys", "description": "Deployment history.",
         "input_schema": {"type": "object", "properties": {"service_name": {"type": "string"}}, "required": ["service_name"]}},
    ]
)

log_analyst = SpecialistAgent(
    name="log-analyst",
    system_prompt="""You are a log analysis expert investigating application errors.

YOUR FOCUS: Error patterns, log frequency, stack traces, correlations.
NOT YOUR FOCUS: Kubernetes infrastructure, runbook procedures.

INVESTIGATION STEPS:
1. Count errors by service to find the hotspot
2. Search for ERROR logs in the affected service
3. Search for WARN logs for precursor signals
4. Get pod logs for detailed stack traces

OUTPUT: Provide a structured log analysis:
- Error frequency and distribution
- Top error types with stack traces
- Warning signals that preceded the errors
- Timeline of error progression
- Correlation between services""",
    tools=[
        {"name": "search_logs", "description": "Search logs by service, level, keyword.",
         "input_schema": {"type": "object", "properties": {"service": {"type": "string"}, "level": {"type": "string", "enum": ["ERROR", "WARN", "INFO"]}, "keyword": {"type": "string"}}}},
        {"name": "count_errors", "description": "Count errors grouped by service.",
         "input_schema": {"type": "object", "properties": {"group_by": {"type": "string", "default": "service"}}}},
        {"name": "get_pod_logs", "description": "Detailed pod logs with stack traces.",
         "input_schema": {"type": "object", "properties": {"pod_name": {"type": "string"}}, "required": ["pod_name"]}},
    ]
)

runbook_advisor = SpecialistAgent(
    name="runbook-advisor",
    system_prompt="""You are an SRE knowledge base advisor.

YOUR FOCUS: Finding relevant runbooks and recommending actions.
NOT YOUR FOCUS: Direct investigation of infrastructure or logs.

Given a problem description, search for relevant runbooks and provide:
1. Which runbooks are relevant
2. Key steps from each runbook
3. Specific commands to execute
4. Escalation guidance if needed

Keep recommendations actionable with copy-paste commands.""",
    tools=[
        {"name": "search_runbooks", "description": "Search SRE runbooks.",
         "input_schema": {"type": "object", "properties": {"query": {"type": "string"}}, "required": ["query"]}},
    ],
    max_iterations=3  # Runbook advisor needs fewer iterations
)


# ============================================================
# TEST: Run specialists independently
# ============================================================

if __name__ == "__main__":
    task = "Payment gateway is degraded with high error rates. Pods seem to be crashing. Investigate the payment-gateway service in the payments namespace."
    
    for agent in [k8s_expert, log_analyst, runbook_advisor]:
        print(f"\n{'=' * 70}")
        print(f"🔍 {agent.name.upper()}")
        print(f"{'=' * 70}")
        
        report = agent.investigate(task)
        
        print(f"\n  Status: {report['status']}")
        print(f"  Iterations: {report['iterations']}")
        print(f"  Tools called: {[tc['tool'] for tc in report['tool_calls']]}")
        print(f"  Tokens: {report['tokens']}")
        print(f"\n  Report:\n{report['report'][:500]}...")
```

---

### Exercise 2: Coordinator Agent (20 min)

**Goal:** Build a coordinator that delegates to specialists, collects their reports, and synthesizes a unified analysis.

Create `week16/coordinator.py`:

```python
"""
Week 16, Exercise 2: Coordinator Agent

The coordinator:
1. Analyzes the task
2. Delegates to specialist agents
3. Collects their reports
4. Synthesizes a unified incident analysis
"""
import anthropic
import json
import time
from concurrent.futures import ThreadPoolExecutor, as_completed

client = anthropic.Anthropic()

# Import specialists from Exercise 1
from specialist_agents import k8s_expert, log_analyst, runbook_advisor, SpecialistAgent


class CoordinatorAgent:
    """
    Manager agent that coordinates specialist agents.
    
    Workflow:
    1. Analyze the incident and determine which specialists to deploy
    2. Run specialists (parallel or sequential)
    3. Collect and synthesize reports
    4. Produce a unified incident analysis
    """
    
    def __init__(self, specialists: list[SpecialistAgent], parallel: bool = True):
        self.specialists = {s.name: s for s in specialists}
        self.parallel = parallel
    
    def investigate(self, task: str) -> dict:
        """Run a coordinated multi-agent investigation."""
        start = time.time()
        
        print(f"\n{'🎯' * 35}")
        print(f"COORDINATED INVESTIGATION")
        print(f"{'🎯' * 35}")
        print(f"\n📋 Task: {task}\n")
        
        # ---- PHASE 1: Determine which specialists to deploy ----
        print("Phase 1: Planning delegation...\n")
        
        delegation = self._plan_delegation(task)
        print(f"  Deploying specialists: {delegation['agents']}")
        for agent_name, subtask in delegation['assignments'].items():
            print(f"    {agent_name}: {subtask[:80]}")
        
        # ---- PHASE 2: Run specialist investigations ----
        print(f"\nPhase 2: Running specialists {'(parallel)' if self.parallel else '(sequential)'}...\n")
        
        reports = {}
        
        if self.parallel:
            reports = self._run_parallel(delegation['assignments'])
        else:
            reports = self._run_sequential(delegation['assignments'])
        
        # ---- PHASE 3: Synthesize reports ----
        print(f"\nPhase 3: Synthesizing reports...\n")
        
        synthesis = self._synthesize(task, reports)
        
        duration = time.time() - start
        
        # Print final analysis
        print(f"\n{'=' * 70}")
        print(f"📋 UNIFIED INCIDENT ANALYSIS (by coordinator)")
        print(f"{'=' * 70}\n")
        print(synthesis['analysis'])
        
        return {
            "analysis": synthesis['analysis'],
            "specialist_reports": {name: r['report'][:200] for name, r in reports.items()},
            "agents_used": list(reports.keys()),
            "total_duration_s": round(duration, 1),
            "total_tokens": sum(r.get('tokens', 0) for r in reports.values()) + synthesis.get('tokens', 0),
        }
    
    def _plan_delegation(self, task: str) -> dict:
        """Use Claude to decide which specialists to deploy and what to ask each."""
        response = client.messages.create(
            model="claude-sonnet-4-5-20250514",
            max_tokens=500,
            temperature=0,
            system="""You are a senior SRE coordinating an incident investigation.
Given a task, decide which specialist agents to deploy and what specific subtask to give each.

Available specialists:
- k8s-expert: Kubernetes pods, deployments, resources, events
- log-analyst: Application logs, error patterns, stack traces
- runbook-advisor: SRE runbooks, procedures, recommended actions

Respond with ONLY valid JSON:
{
  "agents": ["agent-name", ...],
  "assignments": {
    "agent-name": "specific subtask for this agent",
    ...
  }
}""",
            messages=[{"role": "user", "content": task}]
        )
        
        text = response.content[0].text
        # Extract JSON
        start = text.find('{')
        end = text.rfind('}') + 1
        return json.loads(text[start:end])
    
    def _run_parallel(self, assignments: dict) -> dict:
        """Run specialist agents in parallel using threads."""
        reports = {}
        
        def _run_one(agent_name: str, subtask: str):
            agent = self.specialists.get(agent_name)
            if not agent:
                return agent_name, {"agent": agent_name, "status": "unknown_agent", "report": ""}
            
            print(f"  🚀 Starting {agent_name}...")
            start = time.time()
            report = agent.investigate(subtask)
            elapsed = time.time() - start
            print(f"  ✅ {agent_name} done ({elapsed:.1f}s, {report['iterations']} iterations)")
            return agent_name, report
        
        with ThreadPoolExecutor(max_workers=3) as executor:
            futures = {
                executor.submit(_run_one, name, task): name
                for name, task in assignments.items()
            }
            
            for future in as_completed(futures):
                agent_name, report = future.result()
                reports[agent_name] = report
        
        return reports
    
    def _run_sequential(self, assignments: dict) -> dict:
        """Run specialist agents one by one."""
        reports = {}
        for agent_name, subtask in assignments.items():
            agent = self.specialists.get(agent_name)
            if not agent:
                continue
            
            print(f"  🔍 Running {agent_name}...")
            start = time.time()
            report = agent.investigate(subtask)
            elapsed = time.time() - start
            print(f"  ✅ {agent_name} done ({elapsed:.1f}s)")
            reports[agent_name] = report
        
        return reports
    
    def _synthesize(self, original_task: str, reports: dict) -> dict:
        """Synthesize specialist reports into a unified analysis."""
        reports_text = ""
        for name, report in reports.items():
            reports_text += f"\n\n--- {name.upper()} REPORT ---\n{report['report']}"
        
        response = client.messages.create(
            model="claude-sonnet-4-5-20250514",
            max_tokens=2000,
            temperature=0,
            system="""You are a senior SRE synthesizing multiple specialist reports into a unified incident analysis.

Combine the insights from all specialists. Resolve any contradictions.
Cross-reference findings (e.g., K8s OOMKilled + logs showing OutOfMemoryError = confirmed memory issue).

FORMAT:
SEVERITY: P1/P2/P3/P4
INCIDENT SUMMARY: [one paragraph]
ROOT CAUSE: [specific cause with cross-referenced evidence from multiple agents]
EVIDENCE:
  - From K8s Expert: [key findings]
  - From Log Analyst: [key findings]  
  - From Runbook Advisor: [relevant procedures]
IMMEDIATE ACTIONS: [numbered, specific commands]
LONG-TERM FIX: [prevention measures]
COMMUNICATION: [suggested update for stakeholders]""",
            messages=[{
                "role": "user",
                "content": f"Original task: {original_task}\n\nSpecialist reports:{reports_text}\n\nSynthesize into a unified incident analysis."
            }]
        )
        
        return {
            "analysis": response.content[0].text,
            "tokens": response.usage.input_tokens + response.usage.output_tokens
        }


# ============================================================
# RUN COORDINATED INVESTIGATION
# ============================================================

if __name__ == "__main__":
    coordinator = CoordinatorAgent(
        specialists=[k8s_expert, log_analyst, runbook_advisor],
        parallel=False  # Set to True when your environment supports threading with API calls
    )
    
    result = coordinator.investigate(
        "Payment gateway is severely degraded. Error rate is 4.7% and climbing. "
        "Customers are reporting failed transactions. Multiple pods appear to be crashing. "
        "This started about 20 minutes ago. Investigate and provide a complete incident analysis."
    )
    
    print(f"\n{'=' * 70}")
    print("COORDINATION STATS")
    print(f"{'=' * 70}")
    print(f"  Agents used: {result['agents_used']}")
    print(f"  Total duration: {result['total_duration_s']}s")
    print(f"  Total tokens: {result['total_tokens']}")
```

---

### Exercise 3: Agent Communication Patterns (10 min)

**Goal:** Implement a feedback loop where the coordinator can ask specialists for follow-up investigations.

Create `week16/agent_feedback.py`:

```python
"""
Week 16, Exercise 3: Agent Feedback Loops

The coordinator can:
1. Review specialist reports
2. Identify gaps (missing information)
3. Send follow-up requests to specific agents
4. Get refined reports

This produces more thorough investigations for complex incidents.
"""
import anthropic
import json

client = anthropic.Anthropic()


def identify_gaps(original_task: str, reports: dict[str, str]) -> list[dict]:
    """
    Use Claude to identify information gaps in the specialist reports.
    Returns a list of follow-up requests.
    """
    reports_text = "\n\n".join(f"[{name}]: {report[:300]}" for name, report in reports.items())
    
    response = client.messages.create(
        model="claude-sonnet-4-5-20250514",
        max_tokens=500,
        temperature=0,
        system="""You are reviewing specialist investigation reports for completeness.
Identify any gaps — information that was NOT covered but should have been.

Return ONLY valid JSON array of follow-up requests:
[
  {"agent": "agent-name", "request": "specific follow-up question", "priority": "high/medium/low"},
  ...
]

Return [] if no gaps found. Maximum 3 follow-ups.""",
        messages=[{
            "role": "user",
            "content": f"Task: {original_task}\n\nReports:\n{reports_text}\n\nIdentify gaps."
        }]
    )
    
    text = response.content[0].text
    start = text.find('[')
    end = text.rfind(']') + 1
    
    try:
        return json.loads(text[start:end])
    except (json.JSONDecodeError, ValueError):
        return []


class FeedbackCoordinator:
    """
    Coordinator with follow-up capability.
    
    Round 1: All specialists investigate
    Round 2: Coordinator identifies gaps → sends follow-ups
    Round 3: Synthesize everything
    """
    
    def __init__(self, specialists: dict):
        self.specialists = specialists  # name → SpecialistAgent
    
    def investigate_with_feedback(self, task: str, max_followups: int = 2) -> dict:
        """Two-round investigation with feedback."""
        
        # Round 1: Initial investigation
        print("📍 Round 1: Initial specialist investigations...")
        reports = {}
        for name, agent in self.specialists.items():
            print(f"  Running {name}...")
            result = agent.investigate(task)
            reports[name] = result['report']
        
        # Identify gaps
        print("\n📍 Gap analysis...")
        gaps = identify_gaps(task, reports)
        
        if not gaps:
            print("  No gaps found — proceeding to synthesis")
        else:
            print(f"  Found {len(gaps)} gap(s):")
            for gap in gaps:
                print(f"    → [{gap['priority']}] Ask {gap['agent']}: {gap['request'][:60]}")
            
            # Round 2: Follow-up investigations
            print(f"\n📍 Round 2: Follow-up investigations...")
            followup_count = 0
            
            for gap in gaps[:max_followups]:
                agent_name = gap['agent']
                if agent_name in self.specialists:
                    print(f"  Follow-up to {agent_name}: {gap['request'][:60]}...")
                    followup_result = self.specialists[agent_name].investigate(gap['request'])
                    
                    # Append to existing report
                    reports[agent_name] += f"\n\n[FOLLOW-UP]: {followup_result['report']}"
                    followup_count += 1
            
            print(f"  Completed {followup_count} follow-ups")
        
        return {
            "reports": reports,
            "gaps_found": len(gaps),
            "followups_executed": min(len(gaps), max_followups)
        }


# ============================================================
# TEST
# ============================================================

if __name__ == "__main__":
    from specialist_agents import k8s_expert, log_analyst, runbook_advisor
    
    coordinator = FeedbackCoordinator({
        "k8s-expert": k8s_expert,
        "log-analyst": log_analyst,
        "runbook-advisor": runbook_advisor,
    })
    
    result = coordinator.investigate_with_feedback(
        "Payment gateway degraded. Error rate 4.7%. Pods crashing. "
        "Started after a deployment 20 minutes ago."
    )
    
    print(f"\n📊 Investigation summary:")
    print(f"  Gaps found: {result['gaps_found']}")
    print(f"  Follow-ups executed: {result['followups_executed']}")
    for name, report in result['reports'].items():
        print(f"  {name}: {len(report)} chars")
```

---

## 📝 Drills (20 min)

### Drill 1: Parallel Execution Speedup

Modify the coordinator to run specialists in parallel (using `ThreadPoolExecutor`) and measure the speedup:

```python
# Sequential: agent1 (5s) + agent2 (4s) + agent3 (2s) = 11s
# Parallel:   max(agent1, agent2, agent3) = 5s
```

Run 3 investigations both ways and compare wall-clock time.

### Drill 2: Add a 4th Specialist — Deployment Expert

Create a `DeploymentExpert` agent focused on:
- Deployment history and rollback analysis
- Configuration changes (ConfigMaps, Secrets)
- Image version comparison
- Release timeline correlation

Integrate it into the coordinator.

### Drill 3: Agent Voting / Consensus

Have all three specialists independently assess severity (P1-P4). Then compare:

```python
severities = {
    "k8s-expert": "P2",
    "log-analyst": "P1",
    "runbook-advisor": "P2"
}

# Majority vote: P2
# Or: most severe wins for safety: P1
```

Implement both strategies. Which is better for SRE?

### Drill 4: Single vs Multi-Agent Comparison

Run the SAME investigation with:
1. Week 14's single ReAct agent
2. Week 16's multi-agent coordinator

Compare: quality of analysis, total tokens used, wall-clock time, thoroughness of investigation.

Document when single-agent is sufficient vs when multi-agent adds value.

---

## ✅ Week 16 Checklist

Before moving to Week 17:

- [ ] Built 3 specialist agents with narrow scopes and dedicated tools
- [ ] Coordinator agent delegates, collects, and synthesizes reports
- [ ] Understand parallel vs sequential specialist execution
- [ ] Feedback loop identifies gaps and requests follow-up investigations
- [ ] Can articulate when multi-agent > single-agent
- [ ] Know the tradeoffs: complexity, cost, latency, thoroughness

---

## 🧠 Key Concepts from This Week

### Multi-Agent Decision Matrix

```
Scenario                        Best Approach
─────────────────────────       ──────────────
Simple alert triage             Single agent (Week 14)
Service degradation             Single agent with MCP
Multi-service P1 incident       Multi-agent coordinator ✅
Comprehensive daily report      Multi-agent parallel ✅
Quick on-call question          RAG search (Week 8-10)
```

### Cost Comparison

```
Single Agent Investigation:
  ~5 iterations × ~2000 tokens = ~10,000 tokens ≈ $0.03

Multi-Agent Investigation:
  Coordinator: ~3 calls × 2000 tokens = 6,000 tokens
  K8s Expert: ~4 iterations × 1500 = 6,000 tokens  
  Log Analyst: ~3 iterations × 1500 = 4,500 tokens
  Runbook Advisor: ~2 iterations × 1000 = 2,000 tokens
  Synthesis: ~1 call × 3000 = 3,000 tokens
  Total: ~21,500 tokens ≈ $0.07

Multi-agent costs ~2x more but produces deeper, cross-referenced analysis.
Worth it for P1/P2 incidents. Overkill for simple alerts.
```

---

## 🏆 Phase 5 Complete!

You've built the full agent stack:
- Week 14: ReAct agents, planning, guardrails
- Week 15: Persistent memory, error recovery, observability
- Week 16: Multi-agent coordination, specialist delegation, feedback loops

**Portfolio project:** `sre-agent` — autonomous investigation agent with memory, multi-agent coordination, and full observability.

---

*Next week: AI for SRE Operations — applying everything to real SRE workflows: automated incident triage, intelligent alerting, predictive capacity planning, and AI-assisted postmortems.*
