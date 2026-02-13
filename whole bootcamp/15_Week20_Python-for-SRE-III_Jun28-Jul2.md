# PYTHON TRACK — FILE 15 of 48
## PYTHON FOR SRE III
### Automation Scripts, Ansible/Fabric, ChatOps Basics, Scheduling, Putting It All Together
**Week 20 | Jun 28 - Jul 2, 2025 | 5 Days | 1.5 hrs/day**

---

> **THE FINAL PYTHON-FOR-SRE WEEK**
>
> Files 13-14 taught you the building blocks (regex, APIs, concurrency).
> This week you PUT IT ALL TOGETHER into real SRE automation:
>
> - **Automation scripts** that run reliably in production
> - **Remote execution** on multiple servers
> - **ChatOps** — trigger actions from Slack/chat
> - **Scheduling** — cron, APScheduler, daemon patterns
> - **Capstone** — a complete SRE automation toolkit
>
> After this week, you'll have built real tools, not just exercises.

| Day | Topic | Key Skills |
|-----|-------|-----------|
| 1 | Production Automation Scripts | Script patterns, idempotency, dry-run, locking |
| 2 | Remote Execution (Fabric/Paramiko) | SSH from Python, run commands on remote servers |
| 3 | ChatOps & Webhooks | Slack webhooks, bot basics, incident triggers |
| 4 | Scheduling & Daemon Patterns | Cron, APScheduler, long-running services |
| 5 | Capstone: SRE Automation Toolkit | Full project combining everything |

---

## DAY 1: PRODUCTION AUTOMATION SCRIPTS
**Jun 28, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: script patterns, safety features, idempotency
> 00:20-01:10 Hands-on: build production-safe scripts
> 01:10-01:30 Drills: automation challenges

### What Makes a Script "Production-Quality"?

```
A quick script:                    A production script:
- Runs once                        - Runs 1000 times reliably
- Assumes everything works         - Handles every failure
- No logging                       - Logs everything
- No arguments                     - CLI arguments + config
- print("done")                    - Exit codes, error reports
- "Works on my machine"            - Works at 3 AM unattended

This day teaches the patterns that make the difference.
```

### Pattern 1: Script Template

```python
#!/usr/bin/env python3
"""
SRE Automation Script: [description]

Usage:
    python3 script.py --config config.yaml --dry-run
    python3 script.py --config config.yaml --verbose
"""

import argparse
import logging
import sys
import os
import json
from pathlib import Path
from datetime import datetime

# --- Logging Setup ---
def setup_logging(verbose=False, log_file=None):
    level = logging.DEBUG if verbose else logging.INFO
    fmt = "%(asctime)s [%(levelname)-8s] %(name)s: %(message)s"
    
    handlers = [logging.StreamHandler(sys.stderr)]
    if log_file:
        handlers.append(logging.FileHandler(log_file))
    
    logging.basicConfig(level=level, format=fmt, handlers=handlers)
    return logging.getLogger(__name__)

# --- CLI ---
def parse_args():
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("--config", required=True, help="Config file path")
    parser.add_argument("--dry-run", action="store_true", help="Show what would happen")
    parser.add_argument("--verbose", "-v", action="store_true", help="Debug output")
    parser.add_argument("--log-file", help="Log to file")
    return parser.parse_args()

# --- Main Logic ---
def run(config, dry_run=False):
    """Main script logic. Returns True on success."""
    logger = logging.getLogger(__name__)
    
    if dry_run:
        logger.info("DRY RUN — no changes will be made")
    
    # ... your logic here ...
    
    return True

# --- Entry Point ---
def main():
    args = parse_args()
    logger = setup_logging(args.verbose, args.log_file)
    
    logger.info("Script started")
    logger.info(f"Config: {args.config}")
    
    try:
        # Load config
        config = load_config(args.config)
        
        # Run
        success = run(config, dry_run=args.dry_run)
        
        if success:
            logger.info("Script completed successfully")
            sys.exit(0)
        else:
            logger.error("Script completed with errors")
            sys.exit(1)
            
    except KeyboardInterrupt:
        logger.warning("Interrupted by user")
        sys.exit(130)
    except Exception as e:
        logger.exception(f"Script failed: {e}")
        sys.exit(1)

if __name__ == "__main__":
    main()
```

### Pattern 2: Idempotency

```python
# IDEMPOTENT = running the script multiple times has the same effect
# as running it once. This is CRITICAL for production.

# BAD — not idempotent (appends every run):
def bad_add_user(username):
    with open("/etc/users.conf", "a") as f:
        f.write(f"{username}\n")
    # Running 3 times adds 3 copies!

# GOOD — idempotent (check before acting):
def good_add_user(username):
    config = Path("/etc/users.conf").read_text()
    if username in config.split("\n"):
        logger.info(f"User {username} already exists, skipping")
        return False
    with open("/etc/users.conf", "a") as f:
        f.write(f"{username}\n")
    logger.info(f"User {username} added")
    return True

# Pattern: Check → Act → Verify
def ensure_directory(path):
    """Idempotent directory creation."""
    p = Path(path)
    if p.exists():
        if p.is_dir():
            logger.debug(f"Directory exists: {path}")
            return True
        else:
            logger.error(f"Path exists but is not a directory: {path}")
            return False
    p.mkdir(parents=True)
    logger.info(f"Created directory: {path}")
    return True
```

### Pattern 3: Dry Run

```python
class DryRunnable:
    """Mixin for dry-run support."""
    
    def __init__(self, dry_run=False):
        self.dry_run = dry_run
    
    def execute(self, description, func, *args, **kwargs):
        """Execute an action, or just log it in dry-run mode."""
        if self.dry_run:
            logger.info(f"[DRY RUN] Would: {description}")
            return None
        logger.info(f"Executing: {description}")
        return func(*args, **kwargs)

# Usage:
class Deployer(DryRunnable):
    def deploy(self, version):
        self.execute(
            f"Create symlink /opt/app/current → /opt/app/{version}",
            os.symlink, f"/opt/app/{version}", "/opt/app/current"
        )
        self.execute(
            f"Restart service",
            subprocess.run, ["systemctl", "restart", "myapp"]
        )

# Dry run — just shows what WOULD happen:
d = Deployer(dry_run=True)
d.deploy("v2.3.1")
# [DRY RUN] Would: Create symlink /opt/app/current → /opt/app/v2.3.1
# [DRY RUN] Would: Restart service
```

### Pattern 4: File Locking (Prevent Duplicate Runs)

```python
import fcntl
import sys

class FileLock:
    """Prevent multiple instances of a script from running."""
    
    def __init__(self, lock_file="/tmp/my_script.lock"):
        self.lock_file = lock_file
        self.fd = None
    
    def __enter__(self):
        self.fd = open(self.lock_file, "w")
        try:
            fcntl.flock(self.fd, fcntl.LOCK_EX | fcntl.LOCK_NB)
            self.fd.write(str(os.getpid()))
            self.fd.flush()
            return self
        except BlockingIOError:
            print(f"Another instance is already running (lock: {self.lock_file})")
            sys.exit(1)
    
    def __exit__(self, *args):
        fcntl.flock(self.fd, fcntl.LOCK_UN)
        self.fd.close()
        os.remove(self.lock_file)

# Usage:
with FileLock("/tmp/health_checker.lock"):
    # Only one instance can reach here
    run_health_checks()
```

### Pattern 5: Progress and Reporting

```python
import time
import json

class ScriptReport:
    """Track and report script execution."""
    
    def __init__(self, name):
        self.name = name
        self.start_time = time.time()
        self.actions = []
        self.errors = []
        self.warnings = []
    
    def action(self, msg):
        self.actions.append({"time": time.time(), "msg": msg})
        logger.info(msg)
    
    def error(self, msg):
        self.errors.append({"time": time.time(), "msg": msg})
        logger.error(msg)
    
    def warn(self, msg):
        self.warnings.append({"time": time.time(), "msg": msg})
        logger.warning(msg)
    
    def summary(self):
        elapsed = time.time() - self.start_time
        return {
            "script": self.name,
            "duration_seconds": round(elapsed, 2),
            "actions": len(self.actions),
            "errors": len(self.errors),
            "warnings": len(self.warnings),
            "success": len(self.errors) == 0,
        }
    
    def print_summary(self):
        s = self.summary()
        status = "SUCCESS" if s["success"] else "FAILED"
        print(f"\n{'='*50}")
        print(f"Script: {s['script']} [{status}]")
        print(f"Duration: {s['duration_seconds']}s")
        print(f"Actions: {s['actions']}, Errors: {s['errors']}, Warnings: {s['warnings']}")
        print(f"{'='*50}")
```

### DAY 1 EXERCISES

> **EXERCISE 1: Script Template**
>
> Build a reusable script template:
> a) Logging with --verbose and --log-file
> b) --dry-run flag
> c) --config for YAML config
> d) Proper exit codes (0 success, 1 error, 130 interrupt)
> e) Exception handling with traceback logging
> f) Use it as the base for exercises 2 and 3

> **EXERCISE 2: Idempotent Server Setup**
>
> Write a script that sets up a server:
> a) Create required directories (idempotent)
> b) Create config files if they don't exist
> c) Set file permissions
> d) All operations support dry-run
> e) Report: "Created 3 dirs, 2 configs, skipped 4 existing"

> **EXERCISE 3: File Lock + Report**
>
> Build a cleanup script that:
> a) Uses FileLock to prevent duplicate runs
> b) Finds files older than N days in /tmp
> c) Deletes them (with dry-run support)
> d) Generates a ScriptReport with summary
> e) Saves report to JSON

> **DRILL: 10 Automation Pattern Challenges (3 min each)**
>
> 1. Write a script with argparse and proper exit codes
> 2. Implement a dry-run flag
> 3. Make an operation idempotent (check before acting)
> 4. Create a FileLock context manager
> 5. Log to both console and file
> 6. Handle KeyboardInterrupt gracefully
> 7. Load config from YAML with defaults
> 8. Generate a JSON execution report
> 9. Write a function that verifies its own success
> 10. What does idempotent mean? Give 3 examples.

---

## DAY 2: REMOTE EXECUTION
**Jun 29, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: Fabric, Paramiko, remote command execution
> 00:20-01:10 Hands-on: manage remote servers
> 01:10-01:30 Drills: remote automation challenges

### Paramiko — SSH from Python

```python
# Install: pip install paramiko
import paramiko

def run_remote_command(hostname, command, username="sre", key_file=None, timeout=30):
    """Run a command on a remote server via SSH."""
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    
    try:
        connect_kwargs = {
            "hostname": hostname,
            "username": username,
            "timeout": timeout,
        }
        if key_file:
            connect_kwargs["key_filename"] = key_file
        
        client.connect(**connect_kwargs)
        
        stdin, stdout, stderr = client.exec_command(command, timeout=timeout)
        exit_code = stdout.channel.recv_exit_status()
        
        return {
            "hostname": hostname,
            "command": command,
            "stdout": stdout.read().decode().strip(),
            "stderr": stderr.read().decode().strip(),
            "exit_code": exit_code,
            "success": exit_code == 0,
        }
    except paramiko.AuthenticationException:
        return {"hostname": hostname, "error": "Authentication failed", "success": False}
    except paramiko.SSHException as e:
        return {"hostname": hostname, "error": f"SSH error: {e}", "success": False}
    except Exception as e:
        return {"hostname": hostname, "error": str(e), "success": False}
    finally:
        client.close()

# Usage:
result = run_remote_command("web-01", "uptime")
print(result["stdout"])
```

### Fabric — Higher-Level Remote Execution

```python
# Install: pip install fabric
from fabric import Connection, Config
from invoke import Responder

def check_remote_health(hostname, username="sre"):
    """Check health of a remote server."""
    try:
        conn = Connection(hostname, user=username, connect_timeout=10)
        
        # Run commands:
        uptime = conn.run("uptime -p", hide=True).stdout.strip()
        hostname_result = conn.run("hostname", hide=True).stdout.strip()
        
        # Get CPU:
        cpu_result = conn.run(
            "grep 'cpu ' /proc/stat | awk '{usage=($2+$4)*100/($2+$4+$5)} END {printf \"%.1f\", usage}'",
            hide=True
        ).stdout.strip()
        
        # Get memory:
        mem_result = conn.run(
            "free | awk '/Mem:/{printf \"%.1f\", $3/$2*100}'",
            hide=True
        ).stdout.strip()
        
        # Get disk:
        disk_result = conn.run(
            "df -h / | awk 'NR==2{print $5}'",
            hide=True
        ).stdout.strip()
        
        conn.close()
        
        return {
            "hostname": hostname_result,
            "uptime": uptime,
            "cpu_percent": float(cpu_result),
            "memory_percent": float(mem_result),
            "disk_used": disk_result,
            "healthy": True,
        }
    except Exception as e:
        return {"hostname": hostname, "error": str(e), "healthy": False}

# Copy files to remote:
def deploy_config(hostname, local_path, remote_path, username="sre"):
    conn = Connection(hostname, user=username)
    conn.put(local_path, remote_path)
    conn.run(f"chmod 644 {remote_path}")
    conn.close()
```

### Parallel Remote Execution

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import logging

logger = logging.getLogger(__name__)

def run_on_fleet(servers, command, max_workers=10, username="sre"):
    """Run a command on multiple servers in parallel."""
    results = {}
    
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = {
            executor.submit(run_remote_command, server, command, username): server
            for server in servers
        }
        
        for future in as_completed(futures):
            server = futures[future]
            try:
                result = future.result()
                results[server] = result
                status = "✓" if result["success"] else "✗"
                logger.info(f"{status} {server}: {result.get('stdout', result.get('error', ''))[:80]}")
            except Exception as e:
                results[server] = {"hostname": server, "error": str(e), "success": False}
                logger.error(f"✗ {server}: {e}")
    
    return results

# Usage:
servers = ["web-01", "web-02", "web-03", "db-01", "db-02"]
results = run_on_fleet(servers, "uptime -p")

# Summary:
success = sum(1 for r in results.values() if r.get("success"))
print(f"\n{success}/{len(results)} servers responded successfully")
```

### Rolling Deployment Pattern

```python
import time
import logging

logger = logging.getLogger(__name__)

def rolling_deploy(servers, version, batch_size=2, delay_between=30, dry_run=False):
    """Deploy to servers in rolling batches."""
    
    total = len(servers)
    deployed = 0
    failed = []
    
    for i in range(0, total, batch_size):
        batch = servers[i:i + batch_size]
        batch_num = (i // batch_size) + 1
        logger.info(f"Batch {batch_num}: deploying to {batch}")
        
        for server in batch:
            if dry_run:
                logger.info(f"[DRY RUN] Would deploy {version} to {server}")
                deployed += 1
                continue
            
            try:
                # Deploy steps:
                result = run_remote_command(server, f"/opt/app/deploy.sh {version}")
                if result["success"]:
                    deployed += 1
                    logger.info(f"✓ {server}: deployed {version}")
                else:
                    failed.append(server)
                    logger.error(f"✗ {server}: {result.get('stderr', 'unknown error')}")
            except Exception as e:
                failed.append(server)
                logger.error(f"✗ {server}: {e}")
        
        # Health check after batch:
        if not dry_run and i + batch_size < total:
            logger.info(f"Waiting {delay_between}s for health checks...")
            time.sleep(delay_between)
            
            # Check deployed servers are healthy:
            for server in batch:
                health = check_remote_health(server)
                if not health.get("healthy"):
                    logger.error(f"ABORT: {server} unhealthy after deploy!")
                    logger.error(f"Deployed: {deployed}/{total}, Failed: {len(failed)}")
                    return False
    
    logger.info(f"Deployment complete: {deployed}/{total} succeeded, {len(failed)} failed")
    return len(failed) == 0
```

### DAY 2 EXERCISES

> **EXERCISE 1: Remote Health Checker**
>
> Build a tool that checks remote servers:
> a) SSH to each server
> b) Collect: hostname, uptime, CPU, memory, disk
> c) Run in parallel with ThreadPoolExecutor
> d) Output as formatted table
> e) Handle connection failures gracefully

> **EXERCISE 2: Fleet Command Runner**
>
> Build a CLI tool: `fleet-run "command" --servers servers.txt`
> a) Read server list from file
> b) Run command on all servers in parallel
> c) Show results with ✓/✗ status
> d) Support --timeout, --workers, --username flags
> e) Save results to JSON

> **EXERCISE 3: Rolling Deployer (Simulated)**
>
> Simulate a rolling deployment:
> a) 10 servers, batch size 2
> b) Each "deploy" takes 2-5 seconds (random sleep)
> c) 10% chance of failure (random)
> d) Health check between batches
> e) Abort on failure, report progress
> f) Support --dry-run

> **DRILL: 10 Remote Execution Challenges (3 min each)**
>
> 1. Use paramiko to SSH and run a command (or simulate it)
> 2. Handle SSH authentication errors
> 3. Run a command on 5 servers in parallel
> 4. Collect stdout and stderr from remote command
> 5. Copy a file to a remote server (describe the approach)
> 6. What is a rolling deployment?
> 7. Why deploy in batches instead of all at once?
> 8. What should you check between deployment batches?
> 9. Implement a --dry-run for remote commands
> 10. How would you roll back a failed deployment?

---

## DAY 3: CHATOPS & WEBHOOKS
**Jun 30, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: webhooks, Slack integration, ChatOps concepts
> 00:20-01:10 Hands-on: build notification and alert systems
> 01:10-01:30 Drills: webhook challenges

### What Is ChatOps?

```
ChatOps = performing operations through a chat interface.

Instead of:
  1. Get PagerDuty alert
  2. SSH to server
  3. Run diagnostic commands
  4. Copy results
  5. Paste into Slack
  6. Discuss with team

ChatOps:
  1. Bot posts alert to Slack channel
  2. Type: /check web-01
  3. Bot runs diagnostics and posts results
  4. Team discusses with context right there

WHY:
  - Shared context — everyone sees what's happening
  - Audit trail — chat history IS the incident log
  - Speed — trigger actions without leaving Slack
  - Onboarding — new team members learn by watching
```

### Slack Incoming Webhooks

```python
import requests
import json

def send_slack_message(webhook_url, message, channel=None):
    """Send a message to Slack via incoming webhook."""
    payload = {"text": message}
    if channel:
        payload["channel"] = channel
    
    response = requests.post(
        webhook_url,
        json=payload,
        timeout=10,
    )
    response.raise_for_status()
    return response.status_code == 200

# Rich formatting with blocks:
def send_slack_alert(webhook_url, title, fields, severity="warning"):
    """Send a formatted alert to Slack."""
    colors = {"info": "#36a64f", "warning": "#ff9900", "critical": "#ff0000"}
    
    payload = {
        "attachments": [{
            "color": colors.get(severity, "#cccccc"),
            "title": title,
            "fields": [
                {"title": k, "value": str(v), "short": True}
                for k, v in fields.items()
            ],
            "footer": "SRE Monitor",
            "ts": int(time.time()),
        }]
    }
    
    response = requests.post(webhook_url, json=payload, timeout=10)
    return response.ok

# Usage:
SLACK_WEBHOOK = "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

send_slack_alert(
    SLACK_WEBHOOK,
    title="High CPU Alert",
    fields={
        "Server": "web-01",
        "CPU": "94.5%",
        "Memory": "72.0%",
        "Duration": "5 minutes",
    },
    severity="critical",
)
```

### Alert Manager

```python
import requests
import json
import time
import logging
from dataclasses import dataclass, field
from typing import List, Dict

logger = logging.getLogger(__name__)

@dataclass
class Alert:
    title: str
    severity: str                # info, warning, critical
    source: str
    message: str
    fields: Dict[str, str] = field(default_factory=dict)
    timestamp: float = field(default_factory=time.time)

class AlertManager:
    """Manage and route alerts to multiple channels."""
    
    def __init__(self, config):
        self.slack_webhook = config.get("slack_webhook")
        self.pagerduty_key = config.get("pagerduty_key")
        self.alert_history = []
        self.suppression_window = config.get("suppression_minutes", 15) * 60
    
    def send(self, alert):
        """Route alert based on severity."""
        # Check suppression (don't spam):
        if self._is_suppressed(alert):
            logger.debug(f"Suppressed duplicate alert: {alert.title}")
            return
        
        self.alert_history.append(alert)
        
        # Route by severity:
        if alert.severity == "critical":
            self._send_slack(alert)
            self._send_pagerduty(alert)
        elif alert.severity == "warning":
            self._send_slack(alert)
        else:
            logger.info(f"Alert (info): {alert.title}")
    
    def _is_suppressed(self, alert):
        """Don't send duplicate alerts within suppression window."""
        cutoff = time.time() - self.suppression_window
        for prev in self.alert_history:
            if (prev.title == alert.title and
                prev.source == alert.source and
                prev.timestamp > cutoff):
                return True
        return False
    
    def _send_slack(self, alert):
        if not self.slack_webhook:
            logger.warning("Slack webhook not configured")
            return
        try:
            send_slack_alert(
                self.slack_webhook,
                title=alert.title,
                fields={**alert.fields, "source": alert.source},
                severity=alert.severity,
            )
            logger.info(f"Slack alert sent: {alert.title}")
        except Exception as e:
            logger.error(f"Failed to send Slack alert: {e}")
    
    def _send_pagerduty(self, alert):
        """Send to PagerDuty (simplified)."""
        if not self.pagerduty_key:
            logger.warning("PagerDuty not configured")
            return
        # PagerDuty Events API v2:
        payload = {
            "routing_key": self.pagerduty_key,
            "event_action": "trigger",
            "payload": {
                "summary": alert.title,
                "severity": alert.severity,
                "source": alert.source,
                "custom_details": alert.fields,
            }
        }
        try:
            response = requests.post(
                "https://events.pagerduty.com/v2/enqueue",
                json=payload, timeout=10,
            )
            logger.info(f"PagerDuty alert sent: {alert.title}")
        except Exception as e:
            logger.error(f"Failed to send PagerDuty alert: {e}")

# Usage:
config = {
    "slack_webhook": "https://hooks.slack.com/services/...",
    "pagerduty_key": "your-routing-key",
    "suppression_minutes": 15,
}
alerter = AlertManager(config)

alerter.send(Alert(
    title="High CPU on web-01",
    severity="critical",
    source="health-checker",
    message="CPU at 95% for 5 minutes",
    fields={"hostname": "web-01", "cpu": "95%", "duration": "5m"},
))
```

### Webhook Receiver (Simple)

```python
# Receive webhooks (e.g., from Alertmanager, GitHub, PagerDuty)
# Install: pip install flask

from flask import Flask, request, jsonify
import logging

app = Flask(__name__)
logger = logging.getLogger(__name__)

@app.route("/webhook/alert", methods=["POST"])
def receive_alert():
    """Receive an alert webhook and take action."""
    data = request.get_json()
    
    if not data:
        return jsonify({"error": "No JSON body"}), 400
    
    logger.info(f"Received alert: {data.get('title', 'unknown')}")
    
    # Take automated action based on alert:
    alert_type = data.get("type", "")
    
    if alert_type == "high_cpu":
        # Auto-remediation: restart the service
        hostname = data.get("hostname")
        logger.info(f"Auto-remediating: restarting service on {hostname}")
        # run_remote_command(hostname, "systemctl restart myapp")
    
    elif alert_type == "disk_full":
        hostname = data.get("hostname")
        logger.info(f"Auto-remediating: cleaning logs on {hostname}")
        # run_remote_command(hostname, "find /var/log -name '*.gz' -mtime +7 -delete")
    
    return jsonify({"status": "received"}), 200

# Run: python3 -m flask run --port 5000
```

### DAY 3 EXERCISES

> **EXERCISE 1: Slack Alert System**
>
> Build a complete alert notification system:
> a) Send plain text messages to Slack
> b) Send formatted alerts with color-coded severity
> c) Include fields: server, metric, value, threshold
> d) Add suppression (don't send same alert within 15 min)
> e) Test with different severity levels

> **EXERCISE 2: Alert Router**
>
> Build the AlertManager class with:
> a) Slack webhook for all alerts
> b) PagerDuty for critical only (simulate with logging)
> c) Email for daily digest (simulate with logging)
> d) Suppression of duplicate alerts
> e) Alert history with timestamps

> **EXERCISE 3: Webhook Receiver**
>
> Build a Flask webhook endpoint:
> a) Receive JSON POST requests
> b) Validate required fields
> c) Log the alert details
> d) Return appropriate status codes
> e) Trigger a simulated auto-remediation action

> **DRILL: 10 ChatOps Challenges (3 min each)**
>
> 1. Send a message to Slack via webhook (or simulate with requests.post)
> 2. Format a Slack message with attachments
> 3. Build an Alert dataclass with severity, source, message
> 4. Implement alert suppression (dedup within time window)
> 5. What is ChatOps? Why do SRE teams use it?
> 6. What's the difference between incoming and outgoing webhooks?
> 7. How would you trigger a deployment from Slack?
> 8. Route alerts by severity to different channels
> 9. Build a simple Flask endpoint that receives JSON
> 10. What is auto-remediation? Give 3 examples.

---

## DAY 4: SCHEDULING & DAEMON PATTERNS
**Jul 1, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: cron, APScheduler, daemon patterns
> 00:20-01:10 Hands-on: build scheduled tasks
> 01:10-01:30 Drills: scheduling challenges

### Cron — System Scheduling

```bash
# Cron runs scripts on a schedule. Every SRE uses it.

# Edit crontab:
crontab -e

# Format: MIN HOUR DAY MONTH WEEKDAY command
# ┌───── minute (0-59)
# │ ┌───── hour (0-23)
# │ │ ┌───── day of month (1-31)
# │ │ │ ┌───── month (1-12)
# │ │ │ │ ┌───── day of week (0-7, 0 and 7 = Sunday)
# │ │ │ │ │
# * * * * * command

# Examples:
# Every 5 minutes:
*/5 * * * * /opt/sre/health_check.py >> /var/log/health.log 2>&1

# Every hour at :00:
0 * * * * /opt/sre/metrics_collect.py

# Daily at 2 AM:
0 2 * * * /opt/sre/cleanup.py --days 30

# Monday at 9 AM:
0 9 * * 1 /opt/sre/weekly_report.py

# First day of month at midnight:
0 0 1 * * /opt/sre/monthly_report.py
```

### Making Scripts Cron-Safe

```python
#!/usr/bin/env python3
"""
Cron-safe script template.

Cron gotchas to handle:
1. No PATH set — use full paths
2. No terminal — don't use input()
3. No environment — load .env or set in crontab
4. Errors go to email — redirect stderr to log
5. May run before previous finishes — use file locking
"""

import os
import sys
import logging
from pathlib import Path

# Use absolute paths (cron doesn't have your PATH):
SCRIPT_DIR = Path(__file__).parent.resolve()
LOG_DIR = SCRIPT_DIR / "logs"
LOG_DIR.mkdir(exist_ok=True)

# Set up logging to file (cron has no terminal):
logging.basicConfig(
    filename=str(LOG_DIR / "cron_task.log"),
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
)
logger = logging.getLogger(__name__)

# File locking to prevent overlapping runs:
LOCK_FILE = "/tmp/my_cron_task.lock"

def main():
    logger.info("Cron task started")
    # ... your logic ...
    logger.info("Cron task completed")

if __name__ == "__main__":
    # Use the FileLock from Day 1:
    try:
        with FileLock(LOCK_FILE):
            main()
    except SystemExit:
        logger.warning("Another instance already running, skipping")
```

### APScheduler — Python-Native Scheduling

```python
# Install: pip install apscheduler
from apscheduler.schedulers.blocking import BlockingScheduler
from apscheduler.schedulers.background import BackgroundScheduler
import logging
import time

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def check_health():
    logger.info("Running health check...")
    # ... your logic ...

def collect_metrics():
    logger.info("Collecting metrics...")

def send_daily_report():
    logger.info("Sending daily report...")

# Blocking scheduler (for standalone scripts):
scheduler = BlockingScheduler()

# Every 30 seconds:
scheduler.add_job(check_health, "interval", seconds=30)

# Every 5 minutes:
scheduler.add_job(collect_metrics, "interval", minutes=5)

# Daily at 8 AM:
scheduler.add_job(send_daily_report, "cron", hour=8, minute=0)

# Every Monday at 9 AM:
scheduler.add_job(send_daily_report, "cron", day_of_week="mon", hour=9)

try:
    logger.info("Scheduler started")
    scheduler.start()
except KeyboardInterrupt:
    scheduler.shutdown()
    logger.info("Scheduler stopped")
```

### Daemon Pattern (Long-Running Service)

```python
#!/usr/bin/env python3
"""
SRE Monitoring Daemon — runs continuously, performs periodic checks.
"""

import time
import signal
import logging
import sys

logger = logging.getLogger(__name__)

class MonitorDaemon:
    """Long-running monitoring service."""
    
    def __init__(self, check_interval=30):
        self.check_interval = check_interval
        self.running = True
        
        # Handle shutdown signals:
        signal.signal(signal.SIGTERM, self._handle_shutdown)
        signal.signal(signal.SIGINT, self._handle_shutdown)
    
    def _handle_shutdown(self, signum, frame):
        logger.info(f"Received signal {signum}, shutting down gracefully...")
        self.running = False
    
    def run(self):
        logger.info(f"Daemon started (interval: {self.check_interval}s)")
        
        while self.running:
            try:
                self._run_checks()
            except Exception as e:
                logger.exception(f"Check cycle failed: {e}")
            
            # Sleep in small increments (responsive to shutdown):
            for _ in range(self.check_interval):
                if not self.running:
                    break
                time.sleep(1)
        
        logger.info("Daemon stopped")
    
    def _run_checks(self):
        logger.info("Running check cycle...")
        # ... your checks here ...

if __name__ == "__main__":
    daemon = MonitorDaemon(check_interval=30)
    daemon.run()

# Run as a systemd service:
# /etc/systemd/system/sre-monitor.service
# [Unit]
# Description=SRE Monitor Daemon
# After=network.target
#
# [Service]
# ExecStart=/opt/sre/venv/bin/python3 /opt/sre/monitor_daemon.py
# Restart=always
# RestartSec=10
#
# [Install]
# WantedBy=multi-user.target
```

### DAY 4 EXERCISES

> **EXERCISE 1: Cron-Safe Script**
>
> Write a script that:
> a) Checks disk usage on local machine
> b) Logs to a file (not stdout)
> c) Uses file locking
> d) Sends alert if disk > 80%
> e) Write the crontab entry to run it every 5 minutes

> **EXERCISE 2: APScheduler Service**
>
> Build a scheduler that runs:
> a) Health check every 30 seconds
> b) Metrics collection every 5 minutes
> c) Log rotation check every hour
> d) Daily summary report at 8 AM
> e) Handle shutdown gracefully

> **EXERCISE 3: Monitor Daemon**
>
> Build a MonitorDaemon:
> a) Runs health checks on a list of servers
> b) Records results to a log file
> c) Sends alerts when thresholds exceeded
> d) Handles SIGTERM for graceful shutdown
> e) Write a systemd unit file for it

> **DRILL: 10 Scheduling Challenges (3 min each)**
>
> 1. Write a cron expression for "every 5 minutes"
> 2. Write a cron expression for "daily at 3 AM"
> 3. Write a cron expression for "Monday at 9 AM"
> 4. Make a script cron-safe (absolute paths, logging, locking)
> 5. Use APScheduler to run a function every 10 seconds
> 6. Handle SIGTERM in a long-running script
> 7. Why sleep in small increments instead of one big sleep?
> 8. Write a systemd unit file for a Python script
> 9. What's the difference between BlockingScheduler and BackgroundScheduler?
> 10. How do you prevent a cron job from running twice simultaneously?

---

## DAY 5: CAPSTONE — SRE AUTOMATION TOOLKIT
**Jul 2, 2025 | 2 hrs**

> 00:00-00:30 Design: plan the toolkit
> 00:30-01:30 Build: implement all components
> 01:30-02:00 Test and polish

### Capstone Project

> **Build a complete SRE Automation Toolkit that combines EVERYTHING from Files 13-15.**
>
> **Project: `sre-toolkit`**
> ```
> sre-toolkit/
> ├── requirements.txt
> ├── config.yaml
> ├── sre_toolkit/
> │   ├── __init__.py
> │   ├── cli.py              (main CLI with subcommands)
> │   ├── health.py           (health check logic)
> │   ├── fleet.py            (remote execution)
> │   ├── alerts.py           (Slack + PagerDuty alerting)
> │   ├── metrics.py          (Prometheus metrics)
> │   ├── scheduler.py        (scheduled checks)
> │   ├── reports.py          (JSON/CSV report generation)
> │   └── utils.py            (file lock, logging, helpers)
> ├── tests/
> │   ├── test_health.py
> │   ├── test_alerts.py
> │   └── test_reports.py
> └── scripts/
>     ├── health_check.py      (standalone cron script)
>     └── monitor_daemon.py    (long-running daemon)
> ```
>
> **CLI subcommands:**
> ```bash
> # Check health of servers:
> sre-toolkit check web-01 web-02 --timeout 10 --output json
>
> # Run command on fleet:
> sre-toolkit fleet "uptime" --servers servers.txt --workers 20
>
> # Start monitoring daemon:
> sre-toolkit monitor --config config.yaml --interval 30
>
> # Send test alert:
> sre-toolkit alert --severity critical --message "Test alert"
>
> # Generate report:
> sre-toolkit report --from 2025-07-01 --to 2025-07-02 --format csv
> ```
>
> **Requirements checklist:**
> - [ ] CLI with argparse or click (subcommands)
> - [ ] Health checks with parallel execution
> - [ ] Results as text/JSON/CSV
> - [ ] Slack webhook alerts
> - [ ] Prometheus metrics (Counter, Gauge, Histogram)
> - [ ] File locking for cron safety
> - [ ] Logging throughout (not print)
> - [ ] Config from YAML
> - [ ] Dry-run support
> - [ ] Error handling everywhere
> - [ ] At least 5 tests
> - [ ] requirements.txt
> - [ ] README.md

### DAY 5 EXERCISES

> **EXERCISE 1: Build the Capstone**
>
> Follow the project structure above. Implement at minimum:
> a) `check` subcommand with parallel health checks
> b) Alert integration (Slack or logged simulation)
> c) JSON and text output formats
> d) Config from YAML
> e) Proper logging

> **EXERCISE 2: Add Monitoring Daemon**
>
> Add a `monitor` subcommand:
> a) Runs health checks on a schedule
> b) Records results over time
> c) Sends alerts on threshold breach
> d) Handles SIGTERM gracefully
> e) Prometheus metrics exposed

> **EXERCISE 3: Write Tests**
>
> Test all critical paths:
> a) Health check logic (success, timeout, error)
> b) Alert routing (critical → Slack + PD, warning → Slack only)
> c) Report generation (correct format, correct data)
> d) Config loading (valid, missing, defaults)
> e) Run with pytest

---

## WEEK 20 SUMMARY

> **FILE 15 COMPLETE — PYTHON FOR SRE III**
>
> ✓ Production scripts: template, idempotency, dry-run, locking, reporting
> ✓ Remote execution: Paramiko/Fabric SSH, parallel fleet commands
> ✓ Rolling deployments: batch deploys with health checks
> ✓ ChatOps: Slack webhooks, alert routing, suppression
> ✓ Webhook receivers: Flask endpoints for auto-remediation
> ✓ Scheduling: cron, APScheduler, cron-safe scripts
> ✓ Daemon patterns: long-running services, signal handling, systemd
> ✓ Capstone: complete SRE automation toolkit
>
> **INTERVIEW PREP:**
> - "How do you make a script production-quality?" → Logging, error handling,
>   idempotency, dry-run, file locking, proper exit codes, config files
> - "What is idempotency?" → Running multiple times has same effect as once.
>   Check before acting, don't assume clean state.
> - "How do you deploy to many servers?" → Rolling deployment in batches,
>   health check between batches, auto-rollback on failure
> - "What is ChatOps?" → Performing operations through chat. Shared context,
>   audit trail, faster response, team visibility.
> - "How do you schedule Python scripts?" → cron for system-level,
>   APScheduler for Python-native, systemd for daemons
>
> **PYTHON TRACK COMPLETE!** 🎉
> Files 07-15 covered: fundamentals → OOP → error handling → advanced
> (decorators, generators) → modules/venvs/logging → regex/json/csv →
> APIs/concurrency/Prometheus → automation/ChatOps/scheduling
>
> Next: File 16 — Go Fundamentals I (Week 21, Jul 5-9)
> The Go track begins! Go is the language of Kubernetes, Docker, and
> most cloud-native infrastructure tools.
