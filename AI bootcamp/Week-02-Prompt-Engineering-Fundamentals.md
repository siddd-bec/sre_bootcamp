# Week 2: Prompt Engineering Fundamentals

## 🎯 Learning Objectives

By the end of this week, you will:
- Understand and apply zero-shot, few-shot, and chain-of-thought prompting
- Use the CRISP framework to write effective SRE prompts every time
- Leverage XML delimiters (Claude's superpower) to structure your prompts
- Know when each prompting strategy works best
- Build your first few-shot log classifier

---

## 📖 Theory (20 min)

### Why Prompt Engineering Matters

The same LLM can give you a vague, useless answer or a specific, actionable one — the difference is entirely in how you ask. Prompt engineering is the skill of asking well.

**Bad prompt:**
```
What's wrong with my server?
```

**Good prompt:**
```
You are a senior SRE investigating a production issue on a Kubernetes cluster.

The payment-gateway deployment has 3/5 pods in CrashLoopBackOff.
Pod logs show: "java.lang.OutOfMemoryError: Java heap space"
Current memory limit: 512Mi. Pod restarts: 47 in the last hour.

Provide:
1. Root cause analysis
2. Immediate fix (with kubectl commands)
3. Long-term prevention steps
```

The second prompt gives Claude context, role, data, and a clear output format. The answer will be dramatically better.

### The Three Core Prompting Strategies

#### 1. Zero-Shot Prompting

You give the model a task with **no examples**. Just instructions.

```
Classify this log line as ERROR, WARNING, or INFO:
"Connection timed out after 30s to db-primary.prod"
```

**When to use:** Simple, well-defined tasks where the model already understands the format you want.

**Limitation:** The model might format its answer differently each time. One call might return `ERROR`, another might return `This is an ERROR level log because...`

#### 2. Few-Shot Prompting

You give the model **examples** of input → output, then ask it to follow the pattern.

```
Classify each log line:

"Disk usage at 45%" → INFO
"Response time exceeded 2s" → WARNING
"Segmentation fault in worker" → ERROR

Now classify: "Connection timed out after 30s to db-primary.prod" →
```

**When to use:** When you need consistent formatting, or when the task has domain-specific rules the model might not guess correctly.

**Why it works:** The model sees the pattern and replicates it exactly. This is by far the most reliable way to get consistent output.

#### 3. Chain-of-Thought (CoT) Prompting

You ask the model to **think step by step** before giving its answer.

```
Analyze this alert. Think through it step by step before giving your assessment.

Alert: CPU at 95% on web-server-03 for 15 minutes

Step 1: What component is affected?
Step 2: Is 95% CPU critical by itself?
Step 3: Does the 15-minute duration change the severity?
Step 4: What's your overall assessment?
```

**When to use:** Complex problems where the answer requires reasoning — incident analysis, root cause investigation, capacity planning decisions.

**Why it works:** When models "think out loud," they make fewer reasoning errors. It's like showing your work on a math problem.

### The CRISP Framework

A formula for writing great prompts every time:

| Letter | Meaning | What It Does |
|--------|---------|-------------|
| **C** | Context | Background the model needs to understand the situation |
| **R** | Role | Who the model should act as (sets expertise level and perspective) |
| **I** | Instructions | What specifically you want the model to do |
| **S** | Specifics | Constraints, focus areas, edge cases to handle |
| **P** | Pattern | The exact output format you want |

**Example using CRISP:**

```
[C] Our Visa Direct payment platform processes 10,000 TPS on Kubernetes.
    We use PostgreSQL for transaction storage and Redis for rate limiting.

[R] You are a senior SRE incident commander with 10 years of experience.

[I] Analyze this alert and determine if it requires immediate escalation.

[S] Focus on customer impact and payment processing continuity.
    Consider PCI DSS compliance implications.
    Ignore any alerts that are known flappy (listed below).

[P] Respond in this exact format:
    SEVERITY: P1/P2/P3/P4
    CUSTOMER IMPACT: Yes/No — one line description
    ESCALATE: Yes/No
    IMMEDIATE ACTIONS: numbered list of 3-5 steps
    ROOT CAUSE HYPOTHESIS: 2-3 sentences
```

### XML Delimiters — Claude's Secret Weapon

Claude is specifically trained to understand XML tags in prompts. Using them to separate different parts of your prompt dramatically improves accuracy.

```xml
<context>
You are an SRE for a Visa payment processing platform on Kubernetes.
</context>

<incident_data>
Service: payment-gateway
Error rate: 5.2% (normal: 0.01%)
Duration: 12 minutes
Recent deploy: v2.3.1 rolled out 15 minutes ago
</incident_data>

<instructions>
Analyze this incident. Determine if the recent deployment is the likely cause.
</instructions>

<output_format>
LIKELY CAUSE: [deployment / infrastructure / traffic / unknown]
CONFIDENCE: [high / medium / low]
REASONING: [2-3 sentences]
ACTIONS: [numbered list]
</output_format>
```

**Why XML tags help:**
- Clear separation between instructions, data, and format
- The model never confuses your instructions for user data
- You can put data BEFORE instructions without confusing the model
- Makes complex prompts much easier to read and maintain

---

## 🔨 Hands-On (50 min)

### Exercise 1: Zero-Shot vs Few-Shot — Head to Head (20 min)

**Goal:** See the dramatic difference in consistency between zero-shot and few-shot prompting.

Create `week2/zero_vs_few_shot.py`:

```python
"""
Week 2, Exercise 1: Zero-Shot vs Few-Shot Comparison

You'll see:
- Zero-shot gives inconsistent formatting
- Few-shot gives exact, predictable formatting
- Few-shot is essential for building reliable SRE tools
"""
import anthropic

client = anthropic.Anthropic()

# Five log lines to classify
test_logs = [
    "2024-10-15 14:23:01 WARN [PaymentService] Retry attempt 3/5 for tx-9821 - upstream timeout",
    "2024-10-15 14:23:02 ERROR [DBPool] Connection refused to pg-primary:5432 after 3 retries",
    "2024-10-15 14:23:03 INFO [HealthCheck] All endpoints healthy, response time 23ms",
    "2024-10-15 14:23:04 ERROR [AuthService] JWT signature verification failed for token eyJhb...",
    "2024-10-15 14:23:05 WARN [DiskMonitor] /data partition at 87% capacity, threshold is 85%",
]

# ============================================================
# APPROACH 1: Zero-Shot — just ask directly
# ============================================================
print("=" * 70)
print("ZERO-SHOT: Just asking Claude to classify (no examples)")
print("=" * 70)

for log in test_logs:
    response = client.messages.create(
        model="claude-sonnet-4-5-20250514",
        max_tokens=200,
        temperature=0,
        messages=[{
            "role": "user",
            "content": f"Classify this log line by category and severity:\n{log}"
        }]
    )
    # Show the response — notice the formatting varies!
    answer = response.content[0].text.strip()
    print(f"\nLog: {log[:70]}...")
    print(f"Claude: {answer[:150]}")

# ============================================================
# APPROACH 2: Few-Shot — provide examples first
# ============================================================
print(f"\n\n{'=' * 70}")
print("FEW-SHOT: Giving Claude examples to follow (consistent format!)")
print("=" * 70)

few_shot_prompt = """Classify each log line using EXACTLY this format:
Category: NETWORK | DATABASE | APPLICATION | SECURITY | INFRASTRUCTURE
Severity: P1-CRITICAL | P2-HIGH | P3-MEDIUM | P4-LOW
Action: one sentence — what to do right now

Here are examples:

Log: "ERROR [DBPool] All connections exhausted for pool primary-db"
Category: DATABASE
Severity: P1-CRITICAL
Action: Check pg_stat_activity for idle connections and increase pool size

Log: "INFO [Deployer] Deployment auth-service:v1.2.3 completed successfully"
Category: APPLICATION
Severity: P4-LOW
Action: No action needed, verify in monitoring dashboard

Log: "WARN [NetworkPolicy] Packet drop rate 2.3% on eth0 for payment-gateway pods"
Category: NETWORK
Severity: P2-HIGH
Action: Check network policies and node NIC health, compare to baseline

Now classify this log line:
Log: "{log_line}"
"""

for log in test_logs:
    response = client.messages.create(
        model="claude-sonnet-4-5-20250514",
        max_tokens=200,
        temperature=0,
        messages=[{
            "role": "user",
            "content": few_shot_prompt.replace("{log_line}", log)
        }]
    )
    answer = response.content[0].text.strip()
    print(f"\nLog: {log[:70]}...")
    print(f"{answer}")

# ============================================================
# KEY TAKEAWAYS
# ============================================================
print(f"\n\n{'=' * 70}")
print("KEY TAKEAWAYS")
print("=" * 70)
print("""
1. Zero-shot: Each response has different formatting
   → Hard to parse programmatically, unreliable for automation

2. Few-shot: Every response follows the EXACT same format
   → Easy to parse, reliable, production-ready

3. Rule of thumb: If you need consistent output → use few-shot
   If you need quick one-off answers → zero-shot is fine
""")
```

---

### Exercise 2: Chain-of-Thought for Incident Analysis (15 min)

**Goal:** See how step-by-step reasoning produces dramatically deeper analysis.

Create `week2/chain_of_thought.py`:

```python
"""
Week 2, Exercise 2: Chain-of-Thought vs Basic Prompting

For simple questions, CoT is overkill.
For complex analysis (incidents, root cause, capacity), CoT is essential.
"""
import anthropic

client = anthropic.Anthropic()

incident_timeline = """
14:00 - Deploy v2.3.1 of payment-gateway to production (canary at 10%)
14:05 - Canary metrics look normal, error rate 0.01%
14:10 - Full rollout to 100% of pods
14:15 - Alert: Error rate on /api/v1/charge spiked to 5%
14:18 - Alert: P99 latency increased from 200ms to 2.1s
14:20 - Alert: Database connection pool exhaustion on pg-primary-01
14:22 - Customer complaints about failed payments in Zendesk
14:25 - On-call engineer initiates rollback to v2.3.0
14:30 - Error rate returning to baseline (0.02%)
14:35 - All metrics nominal, incident declared resolved
"""

# ============================================================
# APPROACH 1: Basic prompt (no Chain-of-Thought)
# ============================================================
print("=" * 70)
print("BASIC PROMPT (no step-by-step reasoning)")
print("=" * 70)

basic_response = client.messages.create(
    model="claude-sonnet-4-5-20250514",
    max_tokens=800,
    temperature=0,
    messages=[{
        "role": "user",
        "content": f"What caused this incident and what should we do to prevent it?\n\n{incident_timeline}"
    }]
)
print(basic_response.content[0].text)

# ============================================================
# APPROACH 2: Chain-of-Thought with 5 Whys
# ============================================================
print(f"\n\n{'=' * 70}")
print("CHAIN-OF-THOUGHT (structured 5-Whys reasoning)")
print("=" * 70)

cot_response = client.messages.create(
    model="claude-sonnet-4-5-20250514",
    max_tokens=1500,
    temperature=0,
    system=(
        "You are a senior SRE performing root cause analysis. "
        "Think through problems methodically. "
        "Never jump to conclusions — follow the evidence."
    ),
    messages=[{
        "role": "user",
        "content": f"""Perform a root cause analysis of this incident.
Think step by step — do NOT skip to conclusions.

<incident_timeline>
{incident_timeline}
</incident_timeline>

Work through these steps IN ORDER:

STEP 1 — TIMELINE REVIEW
What is the exact sequence of events? Note timestamps and correlations.

STEP 2 — WHAT CHANGED?
What was different right before the problem started? Be specific.

STEP 3 — WHY #1
Why did the error rate spike?

STEP 4 — WHY #2
Why did THAT happen? What's the mechanism?

STEP 5 — WHY #3
What's the deeper underlying cause? Why wasn't this caught in canary?

STEP 6 — ROOT CAUSE
State the fundamental root cause in one sentence.

STEP 7 — ACTION ITEMS
List 5 specific prevention steps.
Each must have: [owner role] — what to do — deadline.
"""
    }]
)
print(cot_response.content[0].text)

# ============================================================
# COMPARISON
# ============================================================
print(f"\n\n{'=' * 70}")
print("COMPARISON")
print("=" * 70)
print(f"  Basic prompt output:  {basic_response.usage.output_tokens} tokens")
print(f"  CoT prompt output:    {cot_response.usage.output_tokens} tokens")
print("""
💡 Key differences:
  - Basic: surface-level answer ("deployment caused it, roll back")
  - CoT: digs into WHY — canary didn't catch it, connection pool issue,
    no load testing for new version, insufficient canary duration
  - CoT produces specific, actionable items with owners and deadlines
  - CoT costs more tokens but quality is dramatically better

When to use CoT:
  ✅ Incident RCA, complex debugging, capacity planning
  ❌ Simple classifications, data extraction, one-line answers
""")
```

---

### Exercise 3: CRISP + XML Delimiters — Production Alert Analyzer (15 min)

**Goal:** Build a real-world prompt using CRISP framework and XML tags.

Create `week2/crisp_alert_analyzer.py`:

```python
"""
Week 2, Exercise 3: CRISP Framework + XML Delimiters

Build a reusable alert analyzer that produces consistent,
structured output for any production alert.
"""
import anthropic

client = anthropic.Anthropic()


def analyze_alert(alert_name, alert_details, service, metrics, recent_changes="None"):
    """
    Analyze a production alert using CRISP + XML structured prompt.
    This function is reusable — call it with any alert data.
    """
    
    prompt = f"""
<context>
Visa Direct payment platform on Kubernetes (EKS). 10,000 TPS.
Stack: payment-gateway, auth-service, fraud-detection, notification-service.
Databases: PostgreSQL primary + 2 replicas, Redis cluster (6 nodes).
SLOs: 99.95% availability, P99 latency < 500ms, error rate < 0.1%.
</context>

<role>
Senior SRE incident commander. Prioritize customer payment impact.
</role>

<alert>
Name: {alert_name}
Service: {service}
Details: {alert_details}
Metrics: {metrics}
Recent Changes: {recent_changes}
</alert>

<instructions>
Analyze this alert. Determine severity, customer impact, and next steps.
Consider whether recent changes are the likely cause.
</instructions>

<rules>
- Known flappy alerts to DEPRIORITIZE: DiskPressure on monitoring nodes, DNS timeouts < 5s
- Payment processing interruption requires VP-level approval
- PCI DSS compliance must be considered for data/security alerts
- Provide specific kubectl/SQL/redis-cli commands — not generic advice
</rules>

<output_format>
SEVERITY: P1-CRITICAL | P2-HIGH | P3-MEDIUM | P4-LOW
CATEGORY: PERFORMANCE | AVAILABILITY | SECURITY | CAPACITY | DEPLOYMENT
CUSTOMER_IMPACT: YES/NO — one line
SLO_BREACH: YES/NO/AT_RISK — which SLO
ESCALATE: YES/NO — to whom
LIKELY_CAUSE: 2-3 sentences
COMMANDS_TO_RUN:
1. [command]
2. [command]
3. [command]
IMMEDIATE_ACTIONS:
1. [action]
2. [action]
3. [action]
</output_format>
"""
    
    response = client.messages.create(
        model="claude-sonnet-4-5-20250514",
        max_tokens=1000,
        temperature=0,
        messages=[{"role": "user", "content": prompt}]
    )
    return response.content[0].text


# ============================================================
# TEST 1: Serious — deployment-caused error spike
# ============================================================
print("=" * 70)
print("ALERT 1: Post-deployment error spike (should be P1/P2)")
print("=" * 70)
print(analyze_alert(
    alert_name="ErrorRateHigh",
    alert_details="HTTP 500 error rate exceeded 2% threshold (current: 4.7%)",
    service="payment-gateway",
    metrics="Error rate: 4.7%, P99: 1.8s, QPS: 9500",
    recent_changes="payment-gateway v2.3.1 deployed 12 minutes ago"
))

# ============================================================
# TEST 2: Capacity warning — not yet critical
# ============================================================
print(f"\n{'=' * 70}")
print("ALERT 2: Database connection warning (should be P2/P3)")
print("=" * 70)
print(analyze_alert(
    alert_name="PostgresConnectionsHigh",
    alert_details="Active connections at 85% of max_connections (170/200)",
    service="pg-primary-01",
    metrics="CPU: 67%, connections: 170/200, replication lag: 0.2s",
    recent_changes="fraud-detection scaled from 5 to 12 pods 30 min ago"
))

# ============================================================
# TEST 3: Known flappy alert — should be deprioritized
# ============================================================
print(f"\n{'=' * 70}")
print("ALERT 3: Known flappy alert (should be P4)")
print("=" * 70)
print(analyze_alert(
    alert_name="DiskPressure",
    alert_details="Node disk pressure on monitoring-node-03",
    service="monitoring infrastructure",
    metrics="Disk: 88%, used by Prometheus TSDB"
))

# ============================================================
# WHAT MADE THIS WORK
# ============================================================
print(f"\n{'=' * 70}")
print("WHY THIS PROMPT IS EFFECTIVE")
print("=" * 70)
print("""
✅ CONTEXT: Claude knows the architecture, scale, and stack
   → Specific commands, not generic advice

✅ ROLE: "Senior SRE incident commander"  
   → Prioritizes customer impact, gives actionable steps

✅ INSTRUCTIONS: Clear task with decision criteria
   → No ambiguity about what we wanted

✅ SPECIFICS (rules): Known flappy alerts, compliance, escalation policy
   → Claude correctly deprioritized DiskPressure on monitoring node

✅ PATTERN: Exact output format with field names
   → Every response parseable, consistent, comparable

✅ XML DELIMITERS: Clean separation
   → Data never confused with instructions
""")
```

---

## 📝 Drills (20 min)

### Drill 1: Prompt Improvement Challenge

Here's a terrible prompt. Rewrite it using CRISP + XML delimiters in a file `week2/drill1_improved.txt`:

**Bad prompt:**
```
Look at these metrics and tell me what's wrong:
CPU 94%, memory 78%, disk 45%, error rate 0.3%, latency p99 890ms
```

Your improved version should:
- Set context (what system, what's normal)
- Set role (SRE, not generic assistant)
- Give clear instructions
- Include specifics (SLO targets, what to ignore)
- Define exact output format

### Drill 2: Kubernetes Event Classifier

Create `week2/drill2_k8s_classifier.py`:

Write a few-shot classifier using these examples:

| Event | Category | Severity |
|-------|----------|----------|
| Pod CrashLoopBackOff | WORKLOAD | P2-HIGH |
| Node NotReady | INFRASTRUCTURE | P1-CRITICAL |
| PVC Bound successfully | STORAGE | P4-LOW |
| HPA scaled from 5→12 replicas | SCALING | P3-MEDIUM |
| Certificate expires in 7 days | SECURITY | P2-HIGH |

Then test with these new events:
- "ImagePullBackOff for container payment-svc"
- "Liveness probe failed for pod auth-svc-abc123"
- "Node memory pressure detected on worker-07"

### Drill 3: Chain-of-Thought Capacity Planning

Create `week2/drill3_capacity.py`:

Write a CoT prompt for this scenario:

```
Current state:
- Payment gateway: 10 pods, each handles 1,000 TPS
- Total platform: 10,000 TPS
- Black Friday in 2 weeks, expected 3x traffic
- Each pod: 2 CPU cores, 4GB memory
- Cluster: 20 nodes, 8 cores and 32GB each
- Pod startup time: 45 seconds
- HPA max replicas: 50

Force step-by-step reasoning:
1. Current capacity utilization
2. Required capacity at 3x
3. Can HPA scale enough?
4. Does the cluster have enough nodes?
5. Bottlenecks?
6. Specific changes needed with timeline
```

### Drill 4: Identify Anti-Patterns

Write `week2/drill4_antipatterns.md` — for each anti-pattern, give a BAD example and a FIXED version:

1. **The Vague Ask** — "Help me with my server"
2. **The Kitchen Sink** — 10 different questions in one prompt
3. **The Missing Context** — Asking about "the error" without details
4. **The Wrong Role** — Wrong expertise for the task
5. **The No-Format** — Analysis without output structure

---

## ✅ Week 2 Checklist

Before moving to Week 3, confirm:

- [ ] Can explain zero-shot vs few-shot vs chain-of-thought
- [ ] Can write a few-shot prompt that gives consistent output
- [ ] Can write a CoT prompt for incident analysis
- [ ] Can apply CRISP to any SRE prompt
- [ ] Can use XML delimiters to structure complex prompts
- [ ] Can identify and fix prompting anti-patterns

---

## 🧠 Strategy Selection Guide

```
Need a quick one-off answer?
  → Zero-shot

Need consistent format for automation?
  → Few-shot (3-5 examples)

Need deep reasoning or analysis?
  → Chain-of-thought

Complex task with specific requirements?
  → CRISP + XML + (few-shot OR CoT)
```

### The Prompt Quality Ladder:

```
Level 1: "What's wrong?"                         ← Useless
Level 2: "Analyze this error: [data]"             ← Okay for chat
Level 3: "Analyze [data]. Format: X / Y / Z"      ← Good for scripts
Level 4: CRISP + XML + examples/CoT               ← Production quality
```

**Always aim for Level 3-4 when building tools.**

---

*Next week: Prompt Engineering for SRE — building a reusable prompt template library for log analysis, incident response, runbook generation, and postmortem writing.*
