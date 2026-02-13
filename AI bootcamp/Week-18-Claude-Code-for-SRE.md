# Week 18: Claude Code for SRE

## 🎯 Learning Objectives

By the end of this week, you will:
- Install and configure Claude Code for your development workflow
- Use Claude Code to generate monitoring scripts, K8s manifests, and Terraform modules
- Master the agentic coding workflow: describe → generate → test → iterate
- Connect Claude Code to your MCP servers for context-aware code generation
- Understand when AI-assisted coding helps vs when to code manually
- Build a collection of SRE automation scripts using Claude Code

---

## 📖 Theory (20 min)

### What is Claude Code?

Claude Code is a **command-line tool** that lets you delegate coding tasks to Claude directly from your terminal. Unlike the chat interface, Claude Code can:

- Read and edit files on your machine
- Run bash commands and see output
- Navigate your codebase and understand project structure
- Use MCP servers for additional context
- Work in an agentic loop: plan → code → test → fix → repeat

```
You (terminal):  "Add Prometheus metrics to the payment service health endpoint"

Claude Code:
  1. Reads your existing code
  2. Plans the changes needed
  3. Writes the code
  4. Runs tests
  5. Fixes any errors
  6. Shows you the diff for approval
```

### Claude Code vs Chat vs API

| Feature | Claude Chat | Claude API | Claude Code |
|---------|------------|-----------|-------------|
| Interface | Web/mobile | Python SDK | Terminal CLI |
| File access | Upload only | None | Full local filesystem |
| Run commands | No | No | Yes (bash, python, etc.) |
| Edit files | No | No | Yes (in-place) |
| MCP servers | Claude Desktop | Manual | Built-in support |
| Best for | Questions, drafts | Automation, pipelines | Coding, file editing |

### When Claude Code Shines for SRE

**Great use cases:**
- Generating boilerplate (K8s manifests, Terraform, CI/CD pipelines)
- Adding observability to existing code (metrics, logging, tracing)
- Writing monitoring scripts with proper error handling
- Converting Splunk queries to ClickHouse SQL
- Writing unit tests for infrastructure code
- Debugging complex bash scripts or Helm charts

**Less useful for:**
- Simple one-line commands you already know
- Security-sensitive code (review everything)
- Performance-critical hot paths (AI code may be suboptimal)
- Code you don't understand (you must be able to review it)

### The Agentic Coding Workflow

```
┌──────────────────────────────────────────────────┐
│ YOU: Describe what you want in natural language    │
└──────────────┬───────────────────────────────────┘
               ↓
┌──────────────────────────────────────────────────┐
│ CLAUDE CODE: Plans approach, reads existing code   │
│ → Writes code → Runs tests → Fixes errors         │
│ → Shows diff for your review                       │
└──────────────┬───────────────────────────────────┘
               ↓
┌──────────────────────────────────────────────────┐
│ YOU: Review, approve/reject, request changes       │
└──────────────────────────────────────────────────┘
```

**Key principle:** Claude Code is a **pair programmer**, not an autonomous developer. You describe, it codes, you review. Always review generated code before committing.

---

## 🔨 Hands-On (50 min)

### Exercise 1: Claude Code Setup & First Tasks (15 min)

**Goal:** Install Claude Code, configure it, and run your first SRE coding tasks.

**Step 1: Install Claude Code**

```bash
# Install via npm (requires Node.js 18+)
npm install -g @anthropic-ai/claude-code

# Verify installation
claude --version

# First run — will prompt for API key
claude
```

**Step 2: Configure with your preferences**

```bash
# Set your preferred editor
claude config set editor vim  # or code, nano, etc.

# Enable verbose mode for learning
claude config set verbose true

# Set cost limit (optional safety)
claude config set maxCostPerTask 1.00
```

**Step 3: First SRE tasks**

Open a terminal in a project directory and try these:

```bash
# Task 1: Generate a health check script
claude "Write a Python health check script that checks:
1. HTTP endpoint responds with 200
2. Response time is under 500ms
3. JSON response contains 'status: healthy'
Include proper error handling, timeout, and retry logic.
Output results as JSON for monitoring integration."

# Task 2: Convert a Splunk query to ClickHouse
claude "Convert this Splunk query to ClickHouse SQL:
index=app_logs sourcetype=payment_gateway level=ERROR 
| stats count by host, message 
| sort -count 
| head 20"

# Task 3: Generate a Kubernetes manifest
claude "Create a Kubernetes deployment manifest for a payment-gateway service:
- 5 replicas
- Image: visa/payment-gw:v2.3.1
- Memory limit: 2Gi, CPU limit: 1000m
- Memory request: 1Gi, CPU request: 500m
- Liveness probe: GET /health on port 8080, delay 30s
- Readiness probe: GET /ready on port 8080, delay 10s
- Anti-affinity: spread across nodes
Include a matching Service (ClusterIP) and HPA (min 3, max 10, target CPU 70%)."
```

Create `week18/first_tasks_log.md` to document what Claude Code generated:

```markdown
# Claude Code First Tasks — Results Log

## Task 1: Health Check Script
**Prompt:** [what you asked]
**Generated file:** health_check.py
**Quality:** [1-5 rating]
**Edits needed:** [what you had to fix]
**Time saved:** [estimate vs writing manually]

## Task 2: Splunk → ClickHouse
**Input query:** [Splunk query]
**Output query:** [ClickHouse SQL]
**Correct?** Yes/No
**Notes:** [any issues]

## Task 3: K8s Manifest
**Generated:** deployment.yaml, service.yaml, hpa.yaml
**Best practices followed?** [anti-affinity, resource limits, probes]
**Production-ready?** Yes/Almost/No
```

---

### Exercise 2: Building SRE Automation Scripts (20 min)

**Goal:** Use Claude Code to build a collection of practical SRE scripts.

For each script below, use Claude Code to generate it, then review and refine.

**Script 1: Pod Resource Analyzer**

```bash
claude "Write a Python script 'pod_analyzer.py' that:
1. Runs 'kubectl top pods -n payments' to get current resource usage
2. Runs 'kubectl get pods -n payments -o json' to get limits/requests
3. Calculates utilization % (usage/limit) for CPU and memory per pod
4. Flags pods above 80% utilization (yellow) or 90% (red)
5. Outputs a clean table with colored indicators
6. If any pod is red, exits with code 1 (for CI/CD integration)

Use subprocess for kubectl calls. Handle kubectl not found gracefully.
Include --namespace and --threshold flags via argparse."
```

**Script 2: Log Pattern Detector**

```bash
claude "Write 'log_detector.py' that reads log lines from stdin and:
1. Detects common error patterns:
   - OutOfMemoryError / OOMKilled
   - Connection refused / timeout
   - 5xx HTTP errors
   - Stack trace sequences (starts with 'at com.' or 'Traceback')
   - Disk full / no space left
2. Groups by pattern type
3. Shows frequency, first/last occurrence, sample message
4. Outputs as JSON for pipeline integration

Usage: kubectl logs deployment/payment-gw | python log_detector.py
Or: cat /var/log/app.log | python log_detector.py"
```

**Script 3: Deployment Safety Checker**

```bash
claude "Write 'deploy_checker.py' that runs pre-deployment safety checks:

1. Health check: all pods in target namespace are Running
2. Error rate: query Prometheus (or simulate) - error rate < 0.5%
3. Recent incidents: no P1/P2 incidents in last 2 hours (check a JSON file)
4. Time check: not deploying on Friday after 3 PM or weekends
5. Change window: within allowed deploy hours (9 AM - 4 PM weekdays)

Output: GO / NO-GO with reasons for each check
Exit code: 0 for GO, 1 for NO-GO

Make it usable as a CI/CD gate:
  python deploy_checker.py --namespace payments --service payment-gateway"
```

**Script 4: On-Call Handoff Generator**

```bash
claude "Write 'handoff_generator.py' that generates an on-call handoff document:

Input: a JSON file with recent incidents, active alerts, and ongoing work items
Output: a markdown document formatted for Slack

Include:
- Current system health (green/yellow/red per service)
- Active incidents summary
- Things to watch (alerts that were noisy but not actionable)
- Upcoming deployments in next 24 hours
- Ongoing maintenance windows

Make it read from a config file 'handoff_config.json' for service list."
```

Create `week18/scripts/` directory and save all generated scripts there. For each one, note in `week18/review_notes.md`:

```markdown
# Script Review Notes

## pod_analyzer.py
- Lines generated: X
- Lines I modified: Y
- Bugs found: [list]
- Quality assessment: [notes]
- Time to generate + review: X min
- Time to write manually (estimate): X min
- **Verdict:** Saved ~X minutes

## log_detector.py
...
```

---

### Exercise 3: Claude Code with MCP Integration (15 min)

**Goal:** Connect Claude Code to your MCP servers so it has context about your infrastructure when generating code.

**Step 1: Configure MCP servers in Claude Code**

Create or edit `~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "sre-tools": {
      "command": "python",
      "args": ["/path/to/week13/sre_mcp_server.py"]
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/your/project"]
    }
  }
}
```

**Step 2: Context-aware code generation**

With MCP connected, Claude Code knows about your infrastructure:

```bash
# Claude Code can now check your actual service health
claude "Check the current health of payment-gateway using the SRE tools,
then write a monitoring script that checks for the specific issues found."

# Claude Code can read your runbooks
claude "Read the OOMKilled recovery runbook from the knowledge base,
then write an automated remediation script that follows those exact steps."

# Claude Code can search your codebase
claude "Look at the existing health check endpoints in our codebase,
then add Prometheus histogram metrics for response latency to each one."
```

**Step 3: Generate a complete monitoring module**

```bash
claude "Using the SRE tools to understand our current services and their health,
create a Python monitoring module 'sre_monitor.py' that:

1. Checks health of all known services every 60 seconds
2. Exports Prometheus metrics:
   - sre_service_health (gauge, labels: service, status)
   - sre_service_error_rate (gauge, labels: service)
   - sre_service_latency_p99 (gauge, labels: service)
   - sre_pod_restarts_total (counter, labels: service, pod)
3. Logs state changes (healthy → degraded) with timestamps
4. Sends a summary to stdout every 5 minutes

Use the prometheus_client library. Make it runnable as a standalone daemon."
```

Create `week18/mcp_integration_notes.md`:

```markdown
# Claude Code + MCP Integration Results

## Did MCP context improve code quality?
- Without MCP: Claude generates generic monitoring code
- With MCP: Claude uses actual service names, actual health fields, actual metrics

## Specific improvements observed:
1. [example]
2. [example]

## Limitations:
- [what didn't work as expected]
```

---

## 📝 Drills (20 min)

### Drill 1: Infrastructure-as-Code Generation

Use Claude Code to generate a complete Terraform module:

```bash
claude "Create a Terraform module for an EKS cluster suitable for SRE workloads:

Module: modules/eks-cluster/
Files: main.tf, variables.tf, outputs.tf

Requirements:
- EKS cluster with managed node groups
- Node group: 3-10 nodes, m5.xlarge
- Enable cluster autoscaler
- Enable CloudWatch logging
- VPC with public and private subnets
- Security groups for pod-to-pod communication
- IAM roles for service accounts (IRSA)
- Tags for cost allocation

Include a README.md with usage example."
```

Review: Does the generated Terraform follow best practices? Any security issues?

### Drill 2: CI/CD Pipeline Generator

```bash
claude "Create a GitHub Actions workflow for deploying a Python service to Kubernetes:

File: .github/workflows/deploy.yaml

Pipeline:
1. Run tests (pytest)
2. Build Docker image
3. Push to ECR
4. Run deploy_checker.py (our safety gate)
5. Deploy to staging (kubectl apply)
6. Run smoke tests against staging
7. Wait for manual approval
8. Deploy to production
9. Monitor for 5 minutes (error rate check)
10. Auto-rollback if error rate > 1%

Include proper secrets management and environment variables."
```

### Drill 3: Refactoring with Claude Code

Take one of your existing bootcamp scripts (e.g., `week14/react_agent.py`) and use Claude Code to refactor it:

```bash
claude "Refactor react_agent.py to:
1. Add type hints to all functions
2. Add docstrings
3. Extract the tool execution into a separate class
4. Add proper logging (replace print statements)
5. Add unit tests in test_react_agent.py
Keep the same functionality but improve code quality."
```

Compare: how does the refactored version compare to the original?

### Drill 4: Claude Code Productivity Measurement

Track your productivity over the 4 exercises:

```markdown
| Task | Manual Estimate | Claude Code Time | Time Saved | Review Effort |
|------|----------------|-----------------|------------|---------------|
| Health check script | 30 min | ? min | ? min | ? min |
| K8s manifests | 45 min | ? min | ? min | ? min |
| Monitoring module | 60 min | ? min | ? min | ? min |
| Terraform module | 90 min | ? min | ? min | ? min |
```

Typical finding: Claude Code generates 80% correct code in 20% of the time. The remaining 20% review/fix effort is where YOUR expertise matters.

---

## ✅ Week 18 Checklist

Before moving to Week 19:

- [ ] Claude Code installed and configured
- [ ] Generated and reviewed 4+ SRE automation scripts
- [ ] Understand the agentic coding workflow (describe → generate → test → iterate)
- [ ] Connected Claude Code to MCP servers for context-aware generation
- [ ] Can articulate when AI-assisted coding helps vs when to code manually
- [ ] Have a collection of SRE scripts as a portfolio deliverable
- [ ] Measured actual productivity gains

---

## 🧠 Key Concepts from This Week

### Claude Code Productivity Model

```
                    ┌─────────────────────────────┐
                    │  YOUR EXPERTISE              │
                    │  (Review, Refine, Decide)    │
                    └──────────┬──────────────────┘
                               │
     ┌─────────────────────────┼─────────────────────────┐
     │                         │                         │
     ↓                         ↓                         ↓
┌──────────┐           ┌──────────┐            ┌──────────┐
│ DESCRIBE │     →     │ GENERATE │     →      │ REVIEW   │
│ (you)    │           │ (Claude) │            │ (you)    │
│ 2 min    │           │ 1 min    │            │ 5-10 min │
└──────────┘           └──────────┘            └──────────┘

Total: ~10-15 min for something that takes 30-60 min manually
Savings: 50-70% of time on boilerplate-heavy tasks
```

### The Review Checklist (Always Do This)

```
Before accepting Claude Code output, check:

□ Security: No hardcoded secrets, proper input validation
□ Error handling: Graceful failures, proper exit codes
□ Logging: Structured, appropriate level, no sensitive data
□ Testing: Edge cases covered, failure modes tested
□ Idempotency: Safe to run multiple times
□ Resource cleanup: No leaked connections, files, processes
□ Dependencies: Minimal, pinned versions
□ Production-ready: Not just "works in demo"
```

### SRE Script Collection (Portfolio Deliverable)

```
week18/scripts/
  ├── health_check.py        — HTTP endpoint health verification
  ├── pod_analyzer.py         — K8s resource utilization analyzer
  ├── log_detector.py         — Log pattern detection pipeline
  ├── deploy_checker.py       — Pre-deployment safety gate
  ├── handoff_generator.py    — On-call handoff document generator
  ├── sre_monitor.py          — Prometheus metrics exporter
  └── README.md               — Collection overview with usage
```

### Interview Angle: AI-Assisted Development

```
"How do you use AI in your daily SRE work?"

"I use Claude Code as a pair programmer for infrastructure automation.
For example, I use it to generate K8s manifests, Terraform modules, and 
monitoring scripts. It handles the boilerplate while I focus on the 
architecture decisions and security review.

I've measured a 50-70% time savings on boilerplate-heavy tasks, while 
maintaining code quality through systematic review. The key insight is 
that AI generates the first draft, but my SRE expertise is what makes 
it production-ready — handling edge cases, security, and operability 
that AI often misses."
```

---

*Next week: AI Platform Engineering — deploy AI services with FastAPI, add Prometheus monitoring, build Grafana dashboards, implement circuit breakers and cost tracking. Making your AI tools production-grade.*
