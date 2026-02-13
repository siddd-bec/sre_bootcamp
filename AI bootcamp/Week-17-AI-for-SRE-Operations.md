# Week 17: AI for SRE Operations

## 🎯 Learning Objectives

By the end of this week, you will:
- Build an automated incident triage pipeline that classifies, prioritizes, and routes alerts
- Create an intelligent alert correlator that reduces alert fatigue by grouping related alerts
- Build an AI-powered postmortem generator from incident timelines
- Implement a capacity planning advisor that analyzes trends and recommends scaling
- Understand which SRE workflows benefit most from AI and which don't
- Have 4 production-ready SRE-AI tools you can demonstrate in interviews

---

## 📖 Theory (20 min)

### Where AI Fits in SRE — And Where It Doesn't

Not every SRE task benefits from AI. Here's a practical framework:

**AI Excels At (use it):**
| Task | Why AI Helps | Traditional Approach |
|------|-------------|---------------------|
| Alert triage & classification | Understands context from log text | Static regex rules |
| Alert correlation | Finds patterns across services | Manual during incidents |
| Postmortem writing | Summarizes timelines into narratives | 2-4 hours of human writing |
| Runbook search | Semantic understanding of problems | Keyword search, tribal knowledge |
| Root cause analysis | Reasons across multiple data sources | Senior engineer intuition |
| Change risk assessment | Cross-references change with history | Manual review checklists |

**AI Struggles With (be cautious):**
| Task | Why AI Struggles | Better Approach |
|------|-----------------|----------------|
| Real-time alerting (< 100ms) | LLM latency too high | Prometheus rules |
| Deterministic threshold checks | Overkill, unreliable for yes/no | Simple comparisons |
| Automated remediation | Risk of wrong action | Human-in-the-loop |
| Compliance/audit decisions | Needs deterministic guarantees | Rule-based systems |

### The AI-SRE Integration Points

```
Alert fires (PagerDuty/OpsGenie)
  ↓
┌─────────────────────────────────────────┐
│ AI TRIAGE LAYER                         │
│                                         │
│ 1. Classify → P1/P2/P3/P4             │
│ 2. Correlate → group related alerts    │
│ 3. Enrich → attach relevant runbooks   │
│ 4. Route → right team, right person    │
│ 5. Summarize → incident brief          │
└─────────────────────────────────────────┘
  ↓
On-call engineer gets:
  "P2 — payment-gateway OOMKilled. 2 pods down.
   Related to deploy v2.3.1 (20 min ago).
   Runbook: increase memory limits.
   Suggested fix: kubectl edit deployment..."

Instead of:
  "FIRING: ErrorRateHigh payment-gateway"
  "FIRING: PodCrashLoopBackOff payment-gw-def34"
  "FIRING: PodCrashLoopBackOff payment-gw-ghi56"
  "FIRING: LatencyP99High payment-gateway"
  (4 separate pages, 3 AM, figuring out they're all the same issue)
```

### Cost of AI in the Alert Path

AI triage adds latency. Here's the budget:

```
Alert fires → AI triage → Engineer notified
             ↑ budget: 5-15 seconds max

Haiku classification (P1/P2/P3/P4):  ~0.5-1s, ~$0.001
Sonnet correlation + enrichment:       ~2-3s, ~$0.01
RAG runbook search:                    ~1-2s, ~free (local)
Total:                                 ~3-6s, ~$0.011

For a team with 50 alerts/day: ~$0.55/day = ~$16/month
That's NOTHING compared to engineer time saved.
```

---

## 🔨 Hands-On (50 min)

### Exercise 1: Intelligent Alert Triage System (20 min)

**Goal:** Build a system that takes raw alerts, classifies severity, identifies root cause, correlates related alerts, and produces an actionable incident brief.

Create `week17/alert_triage.py`:

```python
"""
Week 17, Exercise 1: Intelligent Alert Triage System

Takes raw alerts from monitoring → produces an actionable incident brief.

Pipeline:
  Raw alerts → Classify → Correlate → Enrich with context → Route → Brief
"""
import anthropic
import json
from dataclasses import dataclass, field
from typing import Optional

client = anthropic.Anthropic()


# ============================================================
# ALERT DATA STRUCTURES
# ============================================================

@dataclass
class RawAlert:
    """A raw alert from your monitoring system."""
    alert_name: str
    service: str
    severity: str         # from monitoring (warn/critical)
    message: str
    timestamp: str
    labels: dict = field(default_factory=dict)
    value: Optional[float] = None


@dataclass
class TriagedAlert:
    """An alert after AI triage — enriched with context."""
    original: RawAlert
    ai_severity: str       # P1/P2/P3/P4
    category: str          # outage, degradation, warning, info
    root_cause_hypothesis: str
    affected_users: str    # estimated impact
    correlation_group: str # group ID for related alerts
    recommended_action: str
    runbook_reference: str
    route_to: str          # team to notify


# ============================================================
# STEP 1: Alert Classification
# ============================================================

def classify_alert(alert: RawAlert) -> dict:
    """
    Use Haiku (fast, cheap) to classify an alert.
    Returns: severity, category, impact estimate.
    """
    response = client.messages.create(
        model="claude-haiku-4-5-20241022",
        max_tokens=300,
        temperature=0,
        system="""You classify production alerts for an SRE team. Respond ONLY in JSON.

Severity levels:
  P1: Complete outage, revenue loss, data loss risk. Page immediately.
  P2: Significant degradation affecting many users. Page within 5 min.
  P3: Partial degradation, limited impact. Alert but don't page.
  P4: Warning, no current user impact. Review next business day.

Categories: outage, degradation, resource_pressure, configuration, security, info""",
        messages=[{
            "role": "user",
            "content": f"""Classify this alert:

Alert: {alert.alert_name}
Service: {alert.service}
Monitor severity: {alert.severity}
Message: {alert.message}
Value: {alert.value}
Labels: {json.dumps(alert.labels)}

Return JSON: {{"severity": "P1-P4", "category": "...", "affected_users": "none/few/many/all", "confidence": 0.0-1.0}}"""
        }]
    )
    
    text = response.content[0].text
    start = text.find('{')
    end = text.rfind('}') + 1
    return json.loads(text[start:end])


# ============================================================
# STEP 2: Alert Correlation
# ============================================================

def correlate_alerts(alerts: list[RawAlert]) -> dict[str, list[int]]:
    """
    Group related alerts together.
    Returns: {group_id: [alert_indices]}
    """
    if len(alerts) <= 1:
        return {"group-0": [0]} if alerts else {}
    
    alert_summaries = "\n".join([
        f"[{i}] {a.alert_name} | {a.service} | {a.message[:80]}"
        for i, a in enumerate(alerts)
    ])
    
    response = client.messages.create(
        model="claude-haiku-4-5-20241022",
        max_tokens=300,
        temperature=0,
        system="""You correlate production alerts. Group alerts that are likely caused by the SAME root issue.

Rules:
- Same service + temporally close = likely related
- OOMKilled + CrashLoopBackOff + HighErrorRate = same root cause
- HighLatency in service A + Timeout in service B (which calls A) = related
- Completely independent issues get separate groups

Return ONLY a JSON object: {"group-0": [0, 1, 3], "group-1": [2], "group-2": [4]}
Keys are group IDs, values are alert index numbers.""",
        messages=[{
            "role": "user",
            "content": f"Group these alerts:\n{alert_summaries}"
        }]
    )
    
    text = response.content[0].text
    start = text.find('{')
    end = text.rfind('}') + 1
    return json.loads(text[start:end])


# ============================================================
# STEP 3: Incident Brief Generation
# ============================================================

def generate_incident_brief(alerts: list[RawAlert], classification: dict,
                            correlation_group: str) -> str:
    """
    Generate a concise incident brief for the on-call engineer.
    This is what they see on their phone at 3 AM.
    """
    alert_text = "\n".join([
        f"- [{a.alert_name}] {a.service}: {a.message}" for a in alerts
    ])
    
    response = client.messages.create(
        model="claude-sonnet-4-5-20250514",
        max_tokens=500,
        temperature=0,
        system="""You write incident briefs for on-call SRE engineers.

FORMAT (keep it SHORT — engineer is reading this at 3 AM on their phone):

🔴 P1 — [one-line summary]
Impact: [who is affected]
Root Cause: [likely cause based on alert data]
Action: [specific first step to take]
Runbook: [if applicable]

Be specific. Include service names, error messages, numbers.
No fluff. Every word must be actionable.""",
        messages=[{
            "role": "user",
            "content": f"""Generate incident brief from these correlated alerts:

Alerts:
{alert_text}

AI Classification: {json.dumps(classification)}
Correlation group: {correlation_group}"""
        }]
    )
    
    return response.content[0].text


# ============================================================
# FULL TRIAGE PIPELINE
# ============================================================

def triage_alerts(raw_alerts: list[RawAlert]) -> dict:
    """
    Full triage pipeline:
    1. Classify each alert (parallel in production)
    2. Correlate related alerts into groups
    3. Generate incident brief per group
    """
    print(f"\n{'=' * 70}")
    print(f"ALERT TRIAGE — Processing {len(raw_alerts)} alerts")
    print(f"{'=' * 70}")
    
    # Step 1: Classify
    print(f"\n📊 Step 1: Classifying {len(raw_alerts)} alerts...")
    classifications = []
    for alert in raw_alerts:
        c = classify_alert(alert)
        classifications.append(c)
        print(f"  [{c['severity']}] {alert.alert_name} ({alert.service}) — {c['category']}, affects {c['affected_users']}")
    
    # Step 2: Correlate
    print(f"\n🔗 Step 2: Correlating related alerts...")
    groups = correlate_alerts(raw_alerts)
    
    for group_id, indices in groups.items():
        alert_names = [raw_alerts[i].alert_name for i in indices]
        print(f"  {group_id}: {alert_names}")
    
    # Step 3: Generate brief per group
    print(f"\n📝 Step 3: Generating incident briefs...")
    briefs = {}
    
    for group_id, indices in groups.items():
        group_alerts = [raw_alerts[i] for i in indices]
        
        # Use the highest severity classification in the group
        group_classifications = [classifications[i] for i in indices]
        severity_order = {"P1": 0, "P2": 1, "P3": 2, "P4": 3}
        worst = min(group_classifications, key=lambda c: severity_order.get(c.get("severity", "P4"), 4))
        
        brief = generate_incident_brief(group_alerts, worst, group_id)
        briefs[group_id] = {
            "brief": brief,
            "severity": worst["severity"],
            "alert_count": len(indices),
            "alerts": [raw_alerts[i].alert_name for i in indices]
        }
        
        print(f"\n  --- {group_id} ({worst['severity']}) ---")
        print(f"  {brief}")
    
    return {
        "total_alerts": len(raw_alerts),
        "groups": len(groups),
        "briefs": briefs,
        "classifications": classifications
    }


# ============================================================
# TEST: Simulate an incident with multiple alerts
# ============================================================

if __name__ == "__main__":
    # Simulate: payment-gateway OOM incident generating multiple alerts
    alerts = [
        RawAlert(
            alert_name="ErrorRateHigh",
            service="payment-gateway",
            severity="critical",
            message="Error rate 4.7% exceeds threshold 1%",
            timestamp="2024-10-15T14:23:00Z",
            labels={"namespace": "payments", "team": "platform"},
            value=4.7
        ),
        RawAlert(
            alert_name="PodCrashLoopBackOff",
            service="payment-gateway",
            severity="critical",
            message="Pod payment-gw-def34 in CrashLoopBackOff (47 restarts)",
            timestamp="2024-10-15T14:23:05Z",
            labels={"namespace": "payments", "pod": "payment-gw-def34"},
        ),
        RawAlert(
            alert_name="PodCrashLoopBackOff",
            service="payment-gateway",
            severity="critical",
            message="Pod payment-gw-ghi56 in CrashLoopBackOff (32 restarts)",
            timestamp="2024-10-15T14:23:06Z",
            labels={"namespace": "payments", "pod": "payment-gw-ghi56"},
        ),
        RawAlert(
            alert_name="LatencyP99High",
            service="payment-gateway",
            severity="warning",
            message="P99 latency 2100ms exceeds SLO 500ms",
            timestamp="2024-10-15T14:23:10Z",
            labels={"namespace": "payments"},
            value=2100
        ),
        # Unrelated alert — should be in a different group
        RawAlert(
            alert_name="CertExpiryWarning",
            service="auth-service",
            severity="warning",
            message="TLS certificate expires in 14 days",
            timestamp="2024-10-15T14:00:00Z",
            labels={"namespace": "auth"},
        ),
    ]
    
    result = triage_alerts(alerts)
    
    print(f"\n\n{'=' * 70}")
    print("TRIAGE SUMMARY")
    print(f"{'=' * 70}")
    print(f"  Total alerts: {result['total_alerts']}")
    print(f"  Incident groups: {result['groups']}")
    for gid, info in result['briefs'].items():
        print(f"\n  {gid} [{info['severity']}] — {info['alert_count']} alerts")
        print(f"    Alerts: {info['alerts']}")
```

---

### Exercise 2: AI-Powered Postmortem Generator (15 min)

**Goal:** Generate a structured, blameless postmortem from an incident timeline.

Create `week17/postmortem_generator.py`:

```python
"""
Week 17, Exercise 2: AI-Powered Postmortem Generator

Takes an incident timeline and generates a blameless, 
well-structured postmortem document.

Saves engineers 2-4 hours of writing per incident.
"""
import anthropic
import json

client = anthropic.Anthropic()


def generate_postmortem(incident: dict) -> str:
    """
    Generate a structured postmortem from incident data.
    
    incident format:
    {
        "title": "Payment Gateway OOM Incident",
        "severity": "P2",
        "duration_minutes": 45,
        "services_affected": ["payment-gateway"],
        "timeline": [
            {"time": "14:00", "event": "Deployment v2.3.1 rolled out"},
            {"time": "14:23", "event": "First OOMKilled alert fired"},
            ...
        ],
        "root_cause": "BatchReconciliation loading 2.1M rows into memory",
        "fix_applied": "Increased memory limits to 2Gi, restarted pods",
        "data_points": {
            "error_rate_peak": "4.7%",
            "pods_affected": 2,
            "customer_tickets": 12,
            "revenue_impact": "estimated $3,200 in failed transactions"
        }
    }
    """
    response = client.messages.create(
        model="claude-sonnet-4-5-20250514",
        max_tokens=2500,
        temperature=0.1,  # Slight creativity for better writing
        system="""You write blameless postmortems for SRE teams.

RULES:
1. NEVER blame individuals — focus on systems and processes
2. Use specific data and timestamps
3. Action items must be concrete with owners and deadlines
4. Distinguish between "what happened" and "what we learned"
5. Include both what went well AND what didn't

FORMAT:
# Postmortem: [title]

## Summary
[2-3 sentences: what happened, impact, duration]

## Impact
[Specific metrics: users affected, error rates, revenue, customer tickets]

## Timeline
[Chronological events with timestamps]

## Root Cause Analysis
[Technical explanation of what broke and why. Use 5-Whys.]

## What Went Well
[Things that worked during the incident]

## What Didn't Go Well
[Things that failed or were slow — focus on PROCESSES not people]

## Action Items
[Numbered list with priority, owner placeholder, and deadline]

## Lessons Learned
[Key takeaways for the team]""",
        messages=[{
            "role": "user",
            "content": f"Generate a postmortem from this incident data:\n\n{json.dumps(incident, indent=2)}"
        }]
    )
    
    return response.content[0].text


# ============================================================
# TEST: Generate postmortem from a real incident
# ============================================================

if __name__ == "__main__":
    incident = {
        "title": "Payment Gateway OOM Incident — Oct 15, 2024",
        "severity": "P2",
        "duration_minutes": 45,
        "services_affected": ["payment-gateway"],
        "detection_method": "Automated alert (ErrorRateHigh)",
        "timeline": [
            {"time": "14:00", "event": "Deployment v2.3.1 rolled out via CI/CD. Included new BatchReconciliation daily job."},
            {"time": "14:05", "event": "BatchReconciliation triggered automatically on first run after deployment."},
            {"time": "14:22", "event": "Pods payment-gw-def34 and payment-gw-ghi56 begin Full GC cycles. Heap at 97%."},
            {"time": "14:23", "event": "Both pods OOMKilled. CrashLoopBackOff begins. Error rate spikes to 4.7%."},
            {"time": "14:23", "event": "PagerDuty alert fires: ErrorRateHigh, PodCrashLoopBackOff."},
            {"time": "14:28", "event": "On-call engineer acknowledges alert. Begins investigation."},
            {"time": "14:32", "event": "Root cause identified: BatchReconciliation loading 2.1M transactions into memory at once."},
            {"time": "14:35", "event": "Memory limits increased from 512Mi to 2Gi via kubectl edit deployment."},
            {"time": "14:38", "event": "kubectl rollout restart deployment/payment-gateway executed."},
            {"time": "14:42", "event": "Pods come up successfully. Error rate begins dropping."},
            {"time": "14:50", "event": "Error rate returns to normal (0.02%). Latency back within SLO."},
            {"time": "15:08", "event": "Incident resolved. Monitoring confirmed stable for 15 minutes."},
        ],
        "root_cause": "New BatchReconciliation feature in v2.3.1 loaded all 2.1M daily transactions into an ArrayList in memory. Pod memory limit was 512Mi, insufficient for this data volume. Two of five pods crashed, reducing capacity and increasing error rate for remaining pods.",
        "fix_applied": "Immediate: increased memory limits to 2Gi. Pods restarted successfully.",
        "data_points": {
            "error_rate_peak": "4.7%",
            "error_rate_normal": "0.02%",
            "p99_latency_peak": "2100ms",
            "p99_latency_normal": "180ms",
            "pods_affected": 2,
            "pods_total": 5,
            "pod_restarts": 79,
            "customer_tickets": 12,
            "revenue_impact": "Estimated $3,200 in failed transactions",
            "time_to_detect": "23 minutes (automated)",
            "time_to_mitigate": "12 minutes (from acknowledgment)",
        },
        "participants": ["on-call engineer", "platform team lead (escalated at 14:35)"],
    }
    
    postmortem = generate_postmortem(incident)
    
    print("=" * 70)
    print("GENERATED POSTMORTEM")
    print("=" * 70)
    print(postmortem)
    
    print(f"""
\n💡 This postmortem was generated in ~5 seconds.
   Manually writing it would take 2-4 hours.
   
   The engineer reviews and edits the AI draft (30 min) vs writing from scratch (3 hours).
   
   Time saved per incident: ~2.5 hours
   At 2 incidents/week: 5 hours/week = 260 hours/year saved per team
""")
```

---

### Exercise 3: Capacity Planning Advisor (15 min)

**Goal:** Build an AI advisor that analyzes resource usage trends and recommends scaling actions.

Create `week17/capacity_advisor.py`:

```python
"""
Week 17, Exercise 3: AI Capacity Planning Advisor

Analyzes resource usage trends and provides scaling recommendations.
Combines data analysis with domain knowledge about SRE best practices.
"""
import anthropic
import json

client = anthropic.Anthropic()


def analyze_capacity(service_metrics: dict) -> str:
    """
    Analyze service resource metrics and provide capacity recommendations.
    
    service_metrics format:
    {
        "service": "payment-gateway",
        "current": {
            "pods": 5, "cpu_avg": "65%", "cpu_peak": "88%",
            "memory_avg": "72%", "memory_peak": "95%",
            "disk_usage": "67%", "connections_avg": 85, "connections_max": 200
        },
        "trend_7d": {
            "cpu_avg_change": "+8%", "memory_avg_change": "+12%",
            "traffic_change": "+15%", "error_rate_change": "+0.3%"
        },
        "upcoming": {
            "expected_traffic_increase": "Black Friday — 3x normal traffic in 6 weeks",
            "planned_features": "Adding real-time fraud scoring (memory intensive)"
        },
        "slo": {"error_rate": "< 0.1%", "p99_latency": "< 500ms"}
    }
    """
    response = client.messages.create(
        model="claude-sonnet-4-5-20250514",
        max_tokens=1500,
        temperature=0,
        system="""You are an SRE capacity planning advisor. Analyze resource metrics and provide specific, actionable scaling recommendations.

ANALYSIS FRAMEWORK:
1. Current headroom: How close to limits? (>80% = danger zone)
2. Trend analysis: Growing or stable? When will limits be hit?
3. Upcoming demand: planned events or features that increase load
4. SLO risk: will current trajectory breach SLOs?

RECOMMENDATIONS FORMAT:
🔴 URGENT (act this week): [things that will break soon]
🟡 PLANNED (act this month): [things to schedule]
🟢 WATCH (monitor): [things to keep an eye on]

Include specific numbers: "Scale from 5 to 8 pods" not "increase pods".
Include cost estimates where possible: "Adding 3 pods ≈ +$X/month".
Flag the SLO implications of each recommendation.""",
        messages=[{
            "role": "user",
            "content": f"Analyze capacity for this service:\n\n{json.dumps(service_metrics, indent=2)}"
        }]
    )
    
    return response.content[0].text


def generate_capacity_report(services: list[dict]) -> str:
    """Generate a team-wide capacity report across multiple services."""
    
    # First, analyze each service
    analyses = {}
    for svc in services:
        print(f"  Analyzing {svc['service']}...")
        analyses[svc['service']] = analyze_capacity(svc)
    
    # Then generate summary
    combined = "\n\n---\n\n".join(
        f"## {name}\n{analysis}" for name, analysis in analyses.items()
    )
    
    response = client.messages.create(
        model="claude-sonnet-4-5-20250514",
        max_tokens=800,
        temperature=0,
        system="""You write executive capacity summaries for SRE leadership.
Keep it concise — one paragraph summary + prioritized action list.
Highlight cross-service risks (e.g., if two services are both near limits).""",
        messages=[{
            "role": "user",
            "content": f"Summarize these capacity analyses into an executive report:\n\n{combined}"
        }]
    )
    
    return f"{combined}\n\n---\n\n# Executive Summary\n{response.content[0].text}"


# ============================================================
# TEST
# ============================================================

if __name__ == "__main__":
    services = [
        {
            "service": "payment-gateway",
            "current": {
                "pods": 5, "cpu_avg": "65%", "cpu_peak": "88%",
                "memory_avg": "72%", "memory_peak": "95%",
                "disk_usage": "67%",
                "connections_avg": 85, "connections_max": 200,
                "requests_per_second": 1200
            },
            "trend_7d": {
                "cpu_avg_change": "+8%",
                "memory_avg_change": "+12%",
                "traffic_change": "+15%",
                "error_rate_change": "+0.3%"
            },
            "upcoming": {
                "expected_traffic_increase": "Black Friday in 6 weeks — expected 3x traffic",
                "planned_features": "BatchReconciliation job (memory intensive, runs daily)"
            },
            "slo": {"error_rate": "< 0.1%", "p99_latency": "< 500ms"}
        },
        {
            "service": "auth-service",
            "current": {
                "pods": 4, "cpu_avg": "35%", "cpu_peak": "52%",
                "memory_avg": "45%", "memory_peak": "58%",
                "disk_usage": "23%",
                "connections_avg": 40, "connections_max": 100,
                "requests_per_second": 3000
            },
            "trend_7d": {
                "cpu_avg_change": "+2%",
                "memory_avg_change": "+1%",
                "traffic_change": "+5%",
                "error_rate_change": "0%"
            },
            "upcoming": {
                "expected_traffic_increase": "Black Friday — auth scales linearly with payment traffic",
                "planned_features": "None planned"
            },
            "slo": {"error_rate": "< 0.01%", "p99_latency": "< 100ms"}
        },
        {
            "service": "fraud-detection",
            "current": {
                "pods": 6, "cpu_avg": "78%", "cpu_peak": "93%",
                "memory_avg": "82%", "memory_peak": "91%",
                "disk_usage": "55%",
                "connections_avg": 95, "connections_max": 150,
                "requests_per_second": 800
            },
            "trend_7d": {
                "cpu_avg_change": "+15%",
                "memory_avg_change": "+10%",
                "traffic_change": "+12%",
                "error_rate_change": "+0.05%"
            },
            "upcoming": {
                "expected_traffic_increase": "Black Friday — fraud attempts typically 5x normal",
                "planned_features": "New real-time ML model (2x inference cost)"
            },
            "slo": {"error_rate": "< 0.5%", "p99_latency": "< 300ms"}
        }
    ]
    
    print("=" * 70)
    print("CAPACITY PLANNING REPORT")
    print("=" * 70)
    
    report = generate_capacity_report(services)
    print(report)
```

---

## 📝 Drills (20 min)

### Drill 1: Alert Deduplication

Extend Exercise 1 to deduplicate alerts before triage:

```python
def deduplicate_alerts(alerts: list[RawAlert], time_window_seconds: int = 300) -> list[RawAlert]:
    """
    Deduplicate alerts: if the same alert_name + service fires multiple times
    within the time window, keep only the first occurrence with a count.
    """
```

Test with 20 alerts where 10 are duplicates. Verify triage processes only unique alerts.

### Drill 2: Change Risk Scorer

Build a tool that scores the risk of a proposed production change:

```python
def score_change_risk(change: dict) -> dict:
    """
    Input:
    {
        "service": "payment-gateway",
        "change_type": "deployment",
        "description": "Add batch reconciliation job",
        "files_changed": 12,
        "tests_added": 3,
        "rollback_plan": "kubectl rollout undo",
        "deploy_time": "Tuesday 14:00 EST"
    }
    
    Output:
    {
        "risk_score": 7.5,  # 1-10 scale
        "risk_factors": [...],
        "recommendation": "GO with monitoring" / "DELAY" / "NO-GO",
        "monitoring_checklist": [...]
    }
    """
```

### Drill 3: Incident Communication Drafter

Build a tool that generates stakeholder communications at different levels:

```python
def draft_comms(incident: dict, audience: str) -> str:
    """
    audience options:
    - "engineering": Technical details, specific actions
    - "management": Impact summary, timeline, resolution status  
    - "customer": External-facing, empathetic, no internal details
    """
```

Generate all three for the payment gateway OOM incident. Compare how each version differs.

### Drill 4: Integrate Triage with Your Agent

Connect the alert triage system to your Week 14-16 agent:

1. Alert fires → triage pipeline classifies as P2
2. Agent automatically starts investigation
3. Agent uses MCP tools to gather data
4. Agent produces both an incident brief AND recommended actions

This is a preview of the Week 20 capstone!

---

## ✅ Week 17 Checklist

Before moving to Week 18:

- [ ] Alert triage pipeline: classify → correlate → brief (3 steps, ~5 seconds)
- [ ] AI postmortem generator (saves 2-3 hours per incident)
- [ ] Capacity planning advisor (analyzes trends, recommends specific scaling)
- [ ] Understand where AI fits in SRE vs where traditional tools are better
- [ ] Can estimate cost of AI in the alert path (~$0.01/alert)
- [ ] Have 3 demo-ready SRE-AI tools for interview conversations

---

## 🧠 Key Concepts from This Week

### The AI-SRE Value Matrix

```
                    High AI Value
                         ↑
  Postmortem writing  ●  |  ● Alert triage + correlation
  Capacity planning   ●  |  ● Root cause analysis
  Change risk scoring ●  |  ● Runbook search (RAG)
                         |
  ─────────────────────────────────────── →
                         |        Frequent task
  Log grep            ○  |  ○ Threshold alerting
  Deterministic rules ○  |  ○ Health checks
  Compliance checks   ○  |
                         ↓
                    Low AI Value

●  = Use AI    ○  = Use traditional tools
```

### Interview Story: "I Built an AI Triage System"

```
SITUATION: Our team gets ~50 alerts/day, many are related to the same root 
cause. Engineers spend 10 min per alert just classifying and correlating.

TASK: Reduce time-to-understand from 10 minutes to under 30 seconds.

ACTION: Built an AI triage pipeline:
  - Haiku classifies alerts (P1-P4) in ~1 second, $0.001 each
  - Correlator groups related alerts (reduces 5 pages to 1 incident)
  - Enricher attaches relevant runbooks via RAG
  - Brief generator produces actionable summaries

RESULT: 
  - Time to understand: 10 min → 15 seconds
  - Alert fatigue: 50 pages/day → 12 incident groups/day
  - Cost: ~$16/month
  - Postmortem time: 3 hours → 30 minutes (AI draft + human review)
```

---

*Next week: Claude Code for SRE — using AI-assisted coding for monitoring scripts, Terraform modules, K8s manifests, and CI/CD pipelines.*
