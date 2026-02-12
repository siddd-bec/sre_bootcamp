# Week 3: Prompt Engineering for SRE

## 🎯 Learning Objectives

By the end of this week, you will:
- Build a reusable Python prompt template library for daily SRE work
- Create production-quality templates for: log analysis, incident response, runbook generation, postmortem writing, and change review
- Understand how to parameterize prompts so they work with different inputs
- Chain prompts together (output of one becomes input to the next)
- Have a working `sre_prompts.py` module you'll use throughout the bootcamp

---

## 📖 Theory (20 min)

### Why Build a Template Library?

As an SRE, you'll use AI for the same types of tasks over and over:
- Analyzing log snippets
- Triaging alerts
- Writing incident communications
- Generating runbooks
- Writing postmortems

Instead of writing a new prompt from scratch each time, you build **templates** — prompts with placeholders that you fill in with specific data. Think of them like Jinja templates for infrastructure-as-code, but for AI prompts.

### Anatomy of a Good SRE Prompt Template

```python
template = {
    "name": "incident_response",          # What this template does
    "system": "You are a senior SRE...",   # System prompt (persona + rules)
    "user": """                            # User prompt with {placeholders}
      <incident>
        Service: {service}
        Symptoms: {symptoms}
      </incident>
      Provide your analysis.
    """,
    "model": "claude-sonnet-4-5-20250514", # Which model to use
    "temperature": 0,                       # Always 0 for SRE
    "max_tokens": 1500,                     # Expected output size
}
```

### Prompt Chaining — Multi-Step Workflows

Some tasks are too complex for a single prompt. You break them into steps:

```
Step 1: Classify the alert (few-shot) → severity, category
Step 2: If P1/P2, generate incident analysis (CoT) → root cause hypothesis
Step 3: Generate communication draft (template) → Slack message for stakeholders
```

Each step's output feeds into the next step's input. This is **prompt chaining** — and it's how you build real AI workflows.

### Template Design Principles

1. **One task per template** — Don't try to do everything in one prompt
2. **Clear placeholders** — Use descriptive names: `{service_name}` not `{x}`
3. **Sensible defaults** — Include reasonable defaults where possible
4. **Output validation hint** — Tell the model the exact format so you can parse it
5. **Version your templates** — Track what works and iterate

---

## 🔨 Hands-On (50 min)

### Exercise 1: Build the SRE Prompt Template Library (25 min)

**Goal:** Create a reusable Python module with 5 production-quality templates.

Create `week3/sre_prompts.py`:

```python
"""
Week 3, Exercise 1: SRE Prompt Template Library

This module contains battle-tested prompt templates for common SRE tasks.
Import it into any script: from sre_prompts import SREPrompts

Each template uses:
- CRISP framework (Context, Role, Instructions, Specifics, Pattern)
- XML delimiters for clean separation
- Few-shot examples where consistency matters
- Chain-of-thought where deep reasoning matters
"""
import anthropic
import json
from typing import Optional


class SREPrompts:
    """
    A library of reusable SRE prompt templates.
    
    Usage:
        prompts = SREPrompts()
        result = prompts.analyze_logs(logs="ERROR [db] connection refused...")
        print(result)
    """
    
    def __init__(self, model: str = "claude-sonnet-4-5-20250514"):
        self.client = anthropic.Anthropic()
        self.model = model
    
    def _call(self, system: str, user: str, max_tokens: int = 1500) -> str:
        """Internal: make an API call with standard settings."""
        response = self.client.messages.create(
            model=self.model,
            max_tokens=max_tokens,
            temperature=0,
            system=system,
            messages=[{"role": "user", "content": user}]
        )
        return response.content[0].text
    
    # ================================================================
    # TEMPLATE 1: LOG ANALYSIS
    # Use: Paste a block of logs, get structured analysis
    # Strategy: Few-shot for consistent output format
    # ================================================================
    def analyze_logs(self, logs: str, service: str = "unknown") -> str:
        """
        Analyze a block of log lines for errors, patterns, and anomalies.
        
        Args:
            logs: Raw log text (paste from terminal, file, or ClickHouse)
            service: Name of the service these logs come from
        
        Returns:
            Structured analysis with severity, findings, and actions
        """
        system = """You are an expert SRE log analyst for a payment processing platform.
You analyze logs for: error patterns, performance anomalies, security events, and cascading failures.
You know that log timestamps close together often indicate related events.
You prioritize issues that affect payment processing and customer transactions."""

        user = f"""Analyze these logs from the {service} service.

<examples>
Example input:
"ERROR [DBPool] All connections exhausted for primary-db"
"ERROR [DBPool] All connections exhausted for primary-db"
"WARN [PaymentSvc] Falling back to read replica for tx-123"

Example output:
SEVERITY: P1-CRITICAL
SUMMARY: Database connection pool fully exhausted causing payment fallbacks
KEY_FINDINGS:
1. [DATABASE] Connection pool at 100% — repeated errors indicate sustained exhaustion, not a transient spike
2. [APPLICATION] Payment service is falling back to read replica — this may cause stale reads for transactions
PATTERN: Repeated identical errors suggest the pool is not recovering, likely a connection leak or upstream issue
ACTIONS:
1. Run: SELECT count(*), state FROM pg_stat_activity GROUP BY state — check for idle/stuck connections
2. Check application connection pool settings (max size, idle timeout, leak detection)
3. If connections are stuck: SELECT pg_terminate_backend(pid) for idle connections older than 10 minutes
</examples>

Now analyze these logs:

<logs>
{logs}
</logs>

Respond in the EXACT format shown in the example above.
Include the bracketed category tag before each finding (e.g., [DATABASE], [NETWORK], [SECURITY]).
"""
        return self._call(system, user)
    
    # ================================================================
    # TEMPLATE 2: INCIDENT RESPONSE
    # Use: Active incident — get immediate action plan
    # Strategy: CRISP + CoT for thorough analysis
    # ================================================================
    def incident_response(
        self,
        service: str,
        symptoms: str,
        duration: str,
        customer_impact: str,
        recent_changes: str = "None reported",
        current_metrics: str = "Not provided"
    ) -> str:
        """
        Generate an incident response plan for an active production issue.
        
        Args:
            service: Affected service name
            symptoms: What's happening (errors, latency, etc.)
            duration: How long the issue has been going on
            customer_impact: How customers are affected
            recent_changes: Any recent deployments or config changes
            current_metrics: Current metric values if available
        """
        system = """You are an incident commander for a Tier-1 payment processing platform.
You have managed hundreds of production incidents across distributed systems.
You always consider: customer impact first, then data integrity, then system recovery.
PCI DSS compliance is non-negotiable — any data exposure must be escalated immediately.
You give specific commands, not generic advice."""

        user = f"""We have an active incident. Provide an incident response plan.

Think step by step — assess impact first, then determine actions.

<incident>
Service: {service}
Symptoms: {symptoms}
Duration: {duration}
Customer Impact: {customer_impact}
Recent Changes: {recent_changes}
Current Metrics: {current_metrics}
</incident>

<response_format>
SEVERITY: P1-CRITICAL | P2-HIGH | P3-MEDIUM | P4-LOW
IMPACT_ASSESSMENT: 2-3 sentences on who is affected and how badly

IMMEDIATE_ACTIONS (do these NOW, in this order):
1. [action with specific command or step]
2. [action]
3. [action]

DIAGNOSTIC_COMMANDS (run these to find root cause):
1. [specific command with expected output]
2. [command]
3. [command]

ROOT_CAUSE_HYPOTHESIS: Most likely cause based on symptoms and recent changes.
Explain your reasoning in 3-4 sentences.

COMMUNICATION_DRAFT: A 3-sentence Slack message for #incident-response channel
that covers: what's happening, who's affected, what we're doing about it.

ESCALATION: Who to page if not resolved in 15 minutes.
</response_format>
"""
        return self._call(system, user, max_tokens=2000)
    
    # ================================================================
    # TEMPLATE 3: RUNBOOK GENERATOR
    # Use: Create a step-by-step runbook for a scenario
    # Strategy: Detailed instructions, copy-paste ready commands
    # ================================================================
    def generate_runbook(
        self,
        scenario: str,
        service: str,
        environment: str = "Kubernetes on AWS EKS",
        audience: str = "junior SRE at 3 AM"
    ) -> str:
        """
        Generate a step-by-step runbook for a given failure scenario.
        
        Args:
            scenario: What went wrong (e.g., "Redis cluster node failure")
            service: Which service is affected
            environment: Infrastructure environment
            audience: Who will use this runbook (affects detail level)
        """
        system = f"""You are an SRE documentation expert writing runbooks.
Your runbooks will be used by a {audience} — they must be crystal clear.
Every command must be copy-paste ready with no placeholders left unexplained.
Include expected output for every command so the engineer knows if it worked.
Always include rollback steps and escalation criteria."""

        user = f"""Generate a complete runbook for this scenario.

<scenario>
Failure: {scenario}
Affected Service: {service}
Environment: {environment}
</scenario>

<runbook_format>
# Runbook: [descriptive title]

## Prerequisites
- List of access/tools needed before starting

## Severity Assessment
- How to determine if this is P1 vs P2 vs P3

## Step-by-Step Resolution

### Step 1: [title]
Command: [exact command to run]
Expected output: [what you should see if it worked]
If unexpected: [what to do if output is different]

### Step 2: [title]
(same format)

(continue for all steps)

## Verification
- How to confirm the issue is resolved
- What metrics to watch for 30 minutes after resolution

## Rollback
- If the fix made things worse, how to undo it

## Escalation
- When to escalate (specific criteria)
- Who to contact (role, not name)
- What information to include in the escalation

## Related Runbooks
- Links or references to related procedures
</runbook_format>
"""
        return self._call(system, user, max_tokens=3000)
    
    # ================================================================
    # TEMPLATE 4: POSTMORTEM WRITER
    # Use: After an incident is resolved, generate a blameless postmortem
    # Strategy: CoT to extract lessons, structured output
    # ================================================================
    def write_postmortem(
        self,
        incident_summary: str,
        timeline: str,
        root_cause: str,
        resolution: str,
        impact: str,
        detection_time: str = "Not specified",
        resolution_time: str = "Not specified"
    ) -> str:
        """
        Generate a blameless post-incident review document.
        
        Args:
            incident_summary: Brief description of what happened
            timeline: Chronological events with timestamps
            root_cause: What caused the incident
            resolution: How it was fixed
            impact: Customer and business impact
            detection_time: How long before the issue was detected
            resolution_time: Total time from detection to resolution
        """
        system = """You are writing a blameless post-incident review.
CRITICAL RULES:
- NEVER blame individuals. Use "the team" or "the process" — focus on systemic causes.
- Every action item must be specific, measurable, and have an owner ROLE (not person name).
- "Be more careful" is NOT an action item. "Add automated pre-deploy load test to CI pipeline" IS.
- Focus on what the SYSTEM should do differently, not what PEOPLE should do differently.
- Include what went WELL, not just what went wrong."""

        user = f"""Write a blameless post-incident review.

<incident_data>
Summary: {incident_summary}
Detection Time: {detection_time}
Resolution Time: {resolution_time}
Impact: {impact}
Root Cause: {root_cause}
Resolution: {resolution}

Timeline:
{timeline}
</incident_data>

<postmortem_format>
# Post-Incident Review: [descriptive title]
**Date:** [extract from timeline]
**Severity:** [P1/P2/P3/P4 based on impact]
**Duration:** [detection to resolution]
**Author:** [SRE Team]

## Summary
3-4 sentences covering: what happened, who was affected, how it was resolved.

## Impact
- Customer impact: [specific numbers if available]
- Revenue impact: [estimate if possible]
- SLO impact: [which SLOs were breached and by how much]

## Timeline
[Clean up the provided timeline into a consistent format]
| Time | Event |
|------|-------|

## Root Cause
Explain the root cause in 3-5 sentences. Be specific and technical.
Include the chain of causation (A caused B which caused C).

## Contributing Factors
What other factors made this incident worse or delayed resolution?
List 3-5 contributing factors.

## What Went Well
List 2-3 things that worked correctly during this incident.
(detection, response, communication, etc.)

## Action Items
| # | Action | Owner (role) | Priority | Deadline |
|---|--------|-------------|----------|----------|
| 1 | [specific, measurable action] | [role] | [P1/P2/P3] | [date] |

Include at least 5 action items covering:
- Prevention (stop it from happening again)
- Detection (catch it faster)
- Response (fix it faster)
- Process (improve the workflow)

## Lessons Learned
3-5 key takeaways that the broader team should know.
</postmortem_format>
"""
        return self._call(system, user, max_tokens=3000)
    
    # ================================================================
    # TEMPLATE 5: CHANGE REVIEW
    # Use: Before a production change, assess risk
    # Strategy: Structured risk assessment
    # ================================================================
    def review_change(
        self,
        change_description: str,
        affected_systems: str,
        rollback_plan: str,
        maintenance_window: str = "Not specified",
        testing_done: str = "Not specified"
    ) -> str:
        """
        Review a proposed production change and assess risk.
        
        Args:
            change_description: What is being changed
            affected_systems: Which systems are affected
            rollback_plan: How to undo the change
            maintenance_window: When the change will happen
            testing_done: What testing has been performed
        """
        system = """You are a senior SRE reviewing a proposed production change.
You've seen hundreds of changes go wrong. You know the common failure patterns:
- Config changes that seem safe but affect all pods on restart
- Database migrations that lock tables during peak hours
- Network policy changes that silently break service-to-service communication
- Scaling changes that exhaust resources in other services
You are thorough but not a blocker — you identify risks and suggest mitigations."""

        user = f"""Review this proposed production change.

<change>
Description: {change_description}
Affected Systems: {affected_systems}
Rollback Plan: {rollback_plan}
Maintenance Window: {maintenance_window}
Testing Completed: {testing_done}
</change>

<review_format>
RISK_LEVEL: LOW | MEDIUM | HIGH | CRITICAL
APPROVAL: APPROVED | APPROVED_WITH_CONDITIONS | NEEDS_CHANGES | BLOCKED

BLAST_RADIUS: Which services/users could be affected if this goes wrong?

RISKS_IDENTIFIED:
1. [risk] — Likelihood: HIGH/MEDIUM/LOW — Impact: HIGH/MEDIUM/LOW
   Mitigation: [how to reduce this risk]
2. [risk]
3. [risk]

ROLLBACK_ASSESSMENT: Is the rollback plan adequate? Any gaps?

MONITORING_CHECKLIST:
- [ ] [metric to watch during and after the change]
- [ ] [metric]
- [ ] [metric]

RECOMMENDATIONS:
1. [suggestion to make this change safer]
2. [suggestion]

QUESTIONS_FOR_TEAM:
1. [anything unclear or concerning that needs answers before proceeding]
</review_format>
"""
        return self._call(system, user, max_tokens=1500)


# ================================================================
# CONVENIENCE: Run a template by name with keyword arguments
# ================================================================
def quick_run(template_name: str, **kwargs) -> str:
    """
    Quick-run any template by name.
    
    Usage:
        result = quick_run("analyze_logs", logs="ERROR ...", service="payment-gw")
    """
    prompts = SREPrompts()
    template_func = getattr(prompts, template_name)
    return template_func(**kwargs)
```

---

### Exercise 2: Test Every Template (15 min)

**Goal:** Run each template with realistic SRE data and verify the output quality.

Create `week3/test_templates.py`:

```python
"""
Week 3, Exercise 2: Test all 5 SRE prompt templates

Run each template with realistic data and examine the output.
This also serves as documentation for how to use each template.
"""
from sre_prompts import SREPrompts

prompts = SREPrompts()

# ============================================================
# TEST 1: Log Analysis
# ============================================================
print("=" * 70)
print("TEMPLATE 1: LOG ANALYSIS")
print("=" * 70)

test_logs = """
2024-10-15 14:23:01.234 ERROR [payment-svc] [tx-id=abc123] Connection refused to redis-primary:6379 after 3 retries
2024-10-15 14:23:01.456 ERROR [payment-svc] [tx-id=abc123] Falling back to database for rate limit check
2024-10-15 14:23:01.567 WARN  [auth-svc] [user=batch-sa] JWT token expires in 5 minutes, renewal failed
2024-10-15 14:23:02.890 ERROR [payment-svc] [tx-id=def456] Timeout waiting for response from card-network-gateway (30s)
2024-10-15 14:23:03.123 ERROR [payment-svc] [tx-id=def456] Transaction def456 failed: GATEWAY_TIMEOUT
2024-10-15 14:23:03.456 WARN  [payment-svc] Circuit breaker OPEN for card-network-gateway (failures: 12/10 threshold)
2024-10-15 14:23:04.789 ERROR [payment-svc] [tx-id=ghi789] Circuit breaker rejected request to card-network-gateway
2024-10-15 14:23:05.012 INFO  [monitor] Error rate for payment-svc: 8.3% (threshold: 1%)
"""

result = prompts.analyze_logs(logs=test_logs, service="payment-svc")
print(result)

# ============================================================
# TEST 2: Incident Response
# ============================================================
print(f"\n{'=' * 70}")
print("TEMPLATE 2: INCIDENT RESPONSE")
print("=" * 70)

result = prompts.incident_response(
    service="payment-gateway",
    symptoms="HTTP 500 errors at 8.3%, circuit breaker open to card-network-gateway",
    duration="7 minutes and ongoing",
    customer_impact="~8% of payment transactions failing, customers seeing 'Payment failed, please retry'",
    recent_changes="card-network-gateway TLS cert rotated 20 minutes ago",
    current_metrics="Error rate: 8.3%, P99: 12s (timeout), healthy pods: 5/5, Redis: DOWN"
)
print(result)

# ============================================================
# TEST 3: Runbook Generation
# ============================================================
print(f"\n{'=' * 70}")
print("TEMPLATE 3: RUNBOOK GENERATOR")
print("=" * 70)

result = prompts.generate_runbook(
    scenario="Redis cluster primary node failure causing cache misses and rate limit bypass",
    service="payment-gateway (uses Redis for rate limiting and session cache)",
    environment="Kubernetes on AWS EKS, Redis Cluster 6 nodes (3 primary, 3 replica)"
)
print(result)

# ============================================================
# TEST 4: Postmortem Writer
# ============================================================
print(f"\n{'=' * 70}")
print("TEMPLATE 4: POSTMORTEM")
print("=" * 70)

result = prompts.write_postmortem(
    incident_summary="Payment gateway experienced 8.3% error rate due to Redis failure and card network TLS misconfiguration",
    timeline="""
14:00 - Cert rotation job ran on card-network-gateway
14:03 - Redis primary node crashed (unrelated — OOM)
14:05 - Payment errors begin: Redis cache miss → fallback to DB → slow queries
14:07 - Circuit breaker opens for card-network-gateway (TLS handshake failures)
14:08 - Alert fires: error rate > 1%
14:12 - On-call engineer acknowledges, starts investigation
14:18 - Redis restarted, cache rebuilding
14:22 - Discovered card-network TLS issue — cert chain incomplete
14:28 - Pushed correct cert bundle, circuit breaker reset
14:33 - Error rate back to baseline
14:45 - All clear declared
""",
    root_cause="Two simultaneous issues: Redis OOM crash (insufficient maxmemory config) and incomplete TLS cert chain pushed during rotation",
    resolution="Redis restarted with increased maxmemory. Correct TLS cert bundle deployed to card-network-gateway.",
    impact="~8% of transactions failed for 25 minutes. Estimated 2,400 failed payments. No data loss.",
    detection_time="3 minutes after errors started",
    resolution_time="25 minutes total"
)
print(result)

# ============================================================
# TEST 5: Change Review
# ============================================================
print(f"\n{'=' * 70}")
print("TEMPLATE 5: CHANGE REVIEW")
print("=" * 70)

result = prompts.review_change(
    change_description="Upgrade PostgreSQL from 14.9 to 15.4 on pg-primary-01 (primary database for all payment transactions)",
    affected_systems="payment-gateway, auth-service, fraud-detection, reporting-service — all services that read/write to PostgreSQL",
    rollback_plan="Promote current replica pg-replica-01 (running 14.9) to primary. Point all services to new primary via DNS update.",
    maintenance_window="Saturday 2 AM - 6 AM EST (lowest traffic window)",
    testing_done="Upgrade tested on staging environment. All integration tests passed. pg_upgrade --check passed on production data copy."
)
print(result)
```

---

### Exercise 3: Prompt Chaining — Multi-Step Alert Workflow (10 min)

**Goal:** Chain multiple templates together — the output of one feeds into the next.

Create `week3/prompt_chaining.py`:

```python
"""
Week 3, Exercise 3: Prompt Chaining — Multi-Step Workflow

Real SRE workflow:
  Step 1: Analyze the logs → get severity + findings
  Step 2: If severe, generate incident response plan
  Step 3: Generate a stakeholder communication

Each step's output feeds into the next step.
"""
from sre_prompts import SREPrompts

prompts = SREPrompts()

# ============================================================
# STEP 1: Analyze the logs
# ============================================================
print("STEP 1: Analyzing logs...")
print("=" * 70)

raw_logs = """
2024-10-15 14:23:01 ERROR [payment-svc] Connection refused to redis-primary:6379
2024-10-15 14:23:01 ERROR [payment-svc] Connection refused to redis-primary:6379
2024-10-15 14:23:02 ERROR [payment-svc] Connection refused to redis-primary:6379
2024-10-15 14:23:02 WARN  [payment-svc] Rate limiter fallback: allowing all requests (Redis unavailable)
2024-10-15 14:23:03 ERROR [payment-svc] Transaction tx-001 failed: rate limit check unavailable
2024-10-15 14:23:03 WARN  [fraud-svc] Unable to fetch velocity data from Redis, using stale cache
2024-10-15 14:23:04 ERROR [auth-svc] Session lookup failed: Redis connection refused
"""

log_analysis = prompts.analyze_logs(logs=raw_logs, service="payment-svc")
print(log_analysis)

# ============================================================
# STEP 2: Check if severity warrants incident response
# ============================================================
# Simple check: if "P1" or "P2" appears in the analysis, escalate
is_severe = "P1" in log_analysis or "P2" in log_analysis

if is_severe:
    print(f"\n{'=' * 70}")
    print("STEP 2: Severity is P1/P2 — generating incident response...")
    print("=" * 70)
    
    incident_plan = prompts.incident_response(
        service="payment-svc, auth-svc, fraud-svc",
        symptoms="Redis primary unreachable. Rate limiting disabled. Session lookups failing.",
        duration="~3 minutes and ongoing",
        customer_impact="Rate limits bypassed (security risk), some auth sessions failing",
        recent_changes="None reported",
        current_metrics="Redis: unreachable, Error rate: rising, Fraud checks: degraded"
    )
    print(incident_plan)
    
    # ============================================================
    # STEP 3: Generate stakeholder communication
    # ============================================================
    print(f"\n{'=' * 70}")
    print("STEP 3: Generating stakeholder communication...")
    print("=" * 70)
    
    import anthropic
    client = anthropic.Anthropic()
    
    comms = client.messages.create(
        model="claude-sonnet-4-5-20250514",
        max_tokens=500,
        temperature=0,
        system="You write clear, calm incident communications for technical and non-technical stakeholders.",
        messages=[{
            "role": "user",
            "content": f"""Based on this incident analysis, write TWO communications:

<incident_analysis>
{incident_plan}
</incident_analysis>

1. SLACK MESSAGE for #incident-response (technical audience, 3-4 sentences)
2. EMAIL to VP of Engineering (non-technical, 3-4 sentences, focus on customer impact and ETA)
"""
        }]
    )
    print(comms.content[0].text)

else:
    print("\nSeverity is P3/P4 — no escalation needed.")
    print("Logged for review during business hours.")

# ============================================================
# KEY INSIGHT
# ============================================================
print(f"\n{'=' * 70}")
print("PROMPT CHAINING WORKFLOW")
print("=" * 70)
print("""
  Raw Logs
    ↓
  [Template: analyze_logs] → severity + findings
    ↓
  Decision: Is it P1/P2?
    ↓ YES
  [Template: incident_response] → action plan + diagnostics
    ↓
  [Custom prompt] → stakeholder communications
    ↓
  Engineer has: analysis + plan + comms — all in 30 seconds

💡 This 3-step chain is the foundation of your capstone project (Week 20):
   an AI agent that does all of this AUTOMATICALLY when an alert fires.
""")
```

---

## 📝 Drills (20 min)

### Drill 1: Add a "Health Check Analyzer" Template

Add a 6th template to `sre_prompts.py` called `analyze_health_check`:

```python
def analyze_health_check(self, health_output: str, service: str) -> str:
    """
    Analyze health check endpoint output (JSON) and flag any concerns.
    Input: raw JSON from /health or /readyz endpoint
    Output: status, concerns, recommended actions
    """
    # Write this template yourself!
    # Hint: Use few-shot with examples of healthy vs unhealthy output
```

Test it with this input:
```json
{
  "status": "degraded",
  "checks": {
    "database": {"status": "healthy", "latency_ms": 450},
    "redis": {"status": "unhealthy", "error": "connection refused"},
    "disk": {"status": "healthy", "usage_percent": 72},
    "memory": {"status": "warning", "usage_percent": 89}
  },
  "uptime_seconds": 3600,
  "version": "v2.3.1"
}
```

### Drill 2: Template for ClickHouse Query Generation

Since Visa recently migrated from Splunk to ClickHouse, create a template that takes a natural language question and generates a ClickHouse SQL query:

```python
def generate_clickhouse_query(self, question: str, table_schema: str) -> str:
    """
    Convert a natural language log search question into a ClickHouse SQL query.
    
    Example:
      Input: "Show me all payment errors in the last hour grouped by error type"
      Output: SELECT error_type, count(*) as cnt FROM logs 
              WHERE service = 'payment-svc' AND level = 'ERROR' 
              AND timestamp > now() - INTERVAL 1 HOUR 
              GROUP BY error_type ORDER BY cnt DESC
    """
```

Use few-shot with 3-4 examples of question → query pairs.

### Drill 3: Improve a Template with Better Examples

Take the `analyze_logs` template and improve its few-shot examples. Add:
- An example with a SECURITY finding (e.g., unauthorized access attempt)
- An example with a CASCADING FAILURE (one service causing failures in others)
- An example where the logs show a P4-LOW issue (nothing actionable)

Run the improved template with the same test logs and compare output quality.

### Drill 4: Build a Mini CLI

Create `week3/drill4_cli.py` — a simple command-line interface for the template library:

```bash
# Usage:
python drill4_cli.py logs "ERROR [db] connection refused..."
python drill4_cli.py incident --service payment-gw --symptoms "504 errors at 5%"
python drill4_cli.py runbook --scenario "Redis cluster failure" --service payment-gw
```

Use `sys.argv` or `argparse` to parse the command and arguments, then call the appropriate template.

---

## ✅ Week 3 Checklist

Before moving to Week 4:

- [ ] Have a working `sre_prompts.py` module with 5+ templates
- [ ] Each template uses CRISP + XML delimiters
- [ ] Can call any template with `SREPrompts().method_name(args)`
- [ ] Understand prompt chaining (output of one → input of next)
- [ ] Can add new templates following the established pattern
- [ ] Templates produce consistent, parseable output

---

## 🧠 Key Concepts from This Week

### Template Selection Guide

```
Got raw logs?           → analyze_logs()
Active incident?        → incident_response()
Need a procedure?       → generate_runbook()
Incident resolved?      → write_postmortem()
Proposing a change?     → review_change()
```

### Prompt Chaining Pattern

```
Input → [Template A] → Decision → [Template B] → [Template C] → Output
                          ↓
                    (skip if not severe)
```

### The Library Grows With You

Every time you face a new SRE task, ask: "Could I turn this into a template?"

Good candidates for future templates:
- Capacity planning analysis
- SLO breach investigation
- Dependency failure impact assessment
- Migration risk analysis
- Performance regression analysis
- On-call handoff summary generator

---

*Next week: Advanced Prompting & Structured Output — JSON parsing, production guardrails, retry logic, prompt injection defense. We'll make the template library production-safe.*
