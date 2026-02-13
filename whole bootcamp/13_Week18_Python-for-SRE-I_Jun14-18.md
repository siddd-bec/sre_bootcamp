# PYTHON TRACK — FILE 13 of 48
## PYTHON FOR SRE I
### Regex, JSON/YAML Parsing, CSV Processing, subprocess, os/shutil
**Week 18 | Jun 14 - Jun 18, 2025 | 5 Days | 1.5 hrs/day**

---

> **THE SHIFT: FROM PYTHON TO SRE-PYTHON**
>
> Files 07-12 taught you Python the language.
> Files 13-17 teach you Python FOR SRE WORK.
>
> This week's tools are what you'll use DAILY:
> - Parse logs with **regex** (every SRE does this)
> - Read/write **JSON & YAML** configs
> - Process **CSV** reports and exports
> - Run **shell commands** from Python (subprocess)
> - Manage **files & directories** programmatically
>
> Apple SRE Python interviews test these skills specifically.

| Day | Topic | Key Skills |
|-----|-------|-----------|
| 1 | Regex Fundamentals | re module, patterns, match, search, findall |
| 2 | Regex for Log Parsing | Groups, named groups, real log formats, compile |
| 3 | JSON & YAML | json, yaml, config files, API responses |
| 4 | CSV & Data Processing | csv module, DictReader, data transformation |
| 5 | subprocess & os | Run commands, capture output, file management |

---

## DAY 1: REGEX FUNDAMENTALS
**Jun 14, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: what regex is, basic patterns, re module
> 00:20-01:10 Hands-on: pattern matching exercises
> 01:10-01:30 Drills: rapid pattern practice

### What Is Regex?

```python
# Regular expressions (regex) are PATTERNS for matching text.
# They're the most powerful text search tool you'll ever learn.

# SRE use cases:
# - Parse timestamps from logs
# - Extract IP addresses from configs
# - Validate input formats
# - Find error patterns across millions of log lines
# - Apple SRE interview: "Parse this log with Python"

import re
```

### Basic Patterns

```python
import re

text = "Server web-01 at 10.0.1.1 returned status 500 at 14:30:22"

# LITERAL MATCH — find exact text:
print(re.search("web-01", text))     # Match: 'web-01'
print(re.search("web-99", text))     # None (not found)

# SPECIAL CHARACTERS:
# .     any character (except newline)
# \d    any digit [0-9]
# \w    any word character [a-zA-Z0-9_]
# \s    any whitespace (space, tab, newline)
# \D    any NON-digit
# \W    any NON-word character
# \S    any NON-whitespace

# QUANTIFIERS:
# *     0 or more
# +     1 or more
# ?     0 or 1 (optional)
# {n}   exactly n times
# {n,m} between n and m times

# ANCHORS:
# ^     start of string
# $     end of string
# \b    word boundary
```

### re Module Functions

```python
import re

text = "Server web-01 at 10.0.1.1 returned status 500 at 14:30:22"

# re.search() — find FIRST match anywhere in string:
match = re.search(r"\d+\.\d+\.\d+\.\d+", text)
if match:
    print(match.group())          # '10.0.1.1'

# re.match() — match at the START of string only:
match = re.match(r"Server", text)
if match:
    print(match.group())          # 'Server'

# re.findall() — find ALL matches, return as list:
numbers = re.findall(r"\d+", text)
print(numbers)                     # ['01', '10', '0', '1', '1', '500', '14', '30', '22']

# re.sub() — find and replace:
cleaned = re.sub(r"\d+\.\d+\.\d+\.\d+", "[REDACTED]", text)
print(cleaned)   # Server web-01 at [REDACTED] returned status 500 at 14:30:22

# re.split() — split on pattern:
parts = re.split(r"\s+at\s+", text)
print(parts)     # ['Server web-01', '10.0.1.1 returned status 500', '14:30:22']
```

> **ALWAYS USE RAW STRINGS: r"..."**
>
> In regular Python strings: `\d` means nothing special, but
> `\n` means newline. This creates confusion with regex.
>
> Raw strings (r"...") treat backslashes as literal:
> ```python
> "\\d+"    # regular string — need double backslash
> r"\d+"    # raw string — cleaner, always use this for regex
> ```

### Building Patterns Step by Step

```python
import re

# STRATEGY: Build patterns piece by piece, test each step.

# Example: Match an IP address

# Step 1: One number
r"\d+"                              # matches: 10, 0, 1, 255

# Step 2: Number.Number
r"\d+\.\d+"                         # matches: 10.0, 1.1
# Note: \. means literal dot (. alone means ANY character)

# Step 3: Full IP (4 octets)
r"\d+\.\d+\.\d+\.\d+"              # matches: 10.0.1.1

# Step 4: More precise (1-3 digits per octet)
r"\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}"

# Test it:
text = "Server at 10.0.1.1 and backup at 192.168.1.100"
ips = re.findall(r"\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}", text)
print(ips)                          # ['10.0.1.1', '192.168.1.100']
```

### Common SRE Patterns

```python
import re

# IP address:
ip_pattern = r"\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}"

# Timestamp (2025-06-14 14:30:22):
ts_pattern = r"\d{4}-\d{2}-\d{2}\s+\d{2}:\d{2}:\d{2}"

# HTTP status code:
status_pattern = r"\b[1-5]\d{2}\b"      # \b = word boundary

# Log level:
level_pattern = r"\b(DEBUG|INFO|WARN|WARNING|ERROR|CRITICAL)\b"

# Hostname:
host_pattern = r"\b[a-z]+-\d+\b"        # web-01, db-03, cache-12

# Email:
email_pattern = r"[\w.+-]+@[\w-]+\.[\w.]+"

# Port number:
port_pattern = r":(\d{1,5})\b"

# Key=value pairs:
kv_pattern = r"(\w+)=([\w./-]+)"

# Test:
log = "2025-06-14 14:30:22 ERROR web-01 Connection refused to 10.0.2.1:5432"
print(re.findall(ip_pattern, log))       # ['10.0.2.1']
print(re.findall(ts_pattern, log))       # ['2025-06-14 14:30:22']
print(re.findall(level_pattern, log))    # ['ERROR']
print(re.findall(host_pattern, log))     # ['web-01']
```

### Character Classes

```python
# [abc]    match a, b, or c
# [a-z]    match any lowercase letter
# [A-Z]    match any uppercase letter
# [0-9]    match any digit (same as \d)
# [^abc]   match anything EXCEPT a, b, c
# [a-zA-Z] match any letter

# Examples:
re.findall(r"[A-Z]+", "ERROR on web-01")     # ['ERROR']
re.findall(r"[a-z]+-\d+", "web-01 db-03")    # ['web-01', 'db-03']
re.findall(r"[^0-9]+", "abc123def456")        # ['abc', 'def']
```

### DAY 1 EXERCISES

> **EXERCISE 1: Pattern Matching Practice**
>
> Given this text:
> ```
> Server web-01 (10.0.1.1:8080) responded in 145ms with HTTP 200
> Server web-02 (10.0.1.2:8080) responded in 3200ms with HTTP 503
> Server db-01 (10.0.2.1:5432) responded in 22ms with HTTP 200
> ```
> Write regex to extract:
> a) All server names (web-01, web-02, db-01)
> b) All IP addresses
> c) All port numbers
> d) All response times in ms
> e) All HTTP status codes
> f) Lines with status code >= 500

> **EXERCISE 2: Validate Input**
>
> Write functions using regex:
> a) `is_valid_ip(s)` — True if valid IP format
> b) `is_valid_email(s)` — True if valid email format
> c) `is_valid_hostname(s)` — True if matches pattern like "web-01"
> d) `is_valid_port(s)` — True if number 1-65535
> e) Test each with valid AND invalid inputs

> **EXERCISE 3: Log Cleaner**
>
> Write a function that:
> a) Redacts all IP addresses → [IP_REDACTED]
> b) Redacts all email addresses → [EMAIL_REDACTED]
> c) Redacts anything after "password=" → password=[REDACTED]
> d) Test with sample log lines

> **DRILL: 10 Regex Quickfire (2 min each)**
>
> 1. Match any 3 digits: `\d{3}`
> 2. Match "ERROR" or "WARN": `ERROR|WARN`
> 3. Match a word starting with "web": `\bweb\w*`
> 4. Match a date like 2025-06-14: `\d{4}-\d{2}-\d{2}`
> 5. Find all words in a string: `\w+`
> 6. Match a line that starts with #: `^#`
> 7. Replace multiple spaces with single space: `re.sub(r"\s+", " ", text)`
> 8. Match text inside quotes: `"([^"]+)"`
> 9. Split on comma or semicolon: `re.split(r"[,;]\s*", text)`
> 10. Match a string that's ONLY digits: `^\d+$`

---

## DAY 2: REGEX FOR LOG PARSING
**Jun 15, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: groups, named groups, compile, real log formats
> 00:20-01:10 Hands-on: parse production log formats
> 01:10-01:30 Drills: log parsing challenges

### Capture Groups

```python
import re

# Parentheses () create GROUPS — they capture parts of the match.

log = "2025-06-14 14:30:22 ERROR web-01 Connection refused to 10.0.2.1:5432"

# Without groups — gets everything:
match = re.search(r"\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}", log)
print(match.group())              # '2025-06-14 14:30:22'

# With groups — capture specific parts:
match = re.search(r"(\d{4}-\d{2}-\d{2}) (\d{2}:\d{2}:\d{2})", log)
print(match.group(0))             # '2025-06-14 14:30:22' (full match)
print(match.group(1))             # '2025-06-14' (first group)
print(match.group(2))             # '14:30:22' (second group)
print(match.groups())             # ('2025-06-14', '14:30:22')

# findall with groups returns the GROUP contents:
text = "web-01:8080, web-02:9090, db-01:5432"
pairs = re.findall(r"(\w+-\d+):(\d+)", text)
print(pairs)   # [('web-01', '8080'), ('web-02', '9090'), ('db-01', '5432')]
```

### Named Groups

```python
import re

# Named groups: (?P<name>pattern) — much more readable!

log = "2025-06-14 14:30:22 ERROR web-01 Connection refused to 10.0.2.1:5432"

pattern = r"(?P<date>\d{4}-\d{2}-\d{2}) (?P<time>\d{2}:\d{2}:\d{2}) (?P<level>\w+) (?P<host>\S+) (?P<message>.*)"

match = re.search(pattern, log)
if match:
    print(match.group("date"))     # '2025-06-14'
    print(match.group("level"))    # 'ERROR'
    print(match.group("host"))     # 'web-01'
    print(match.group("message"))  # 'Connection refused to 10.0.2.1:5432'

    # Convert to dict (very useful!):
    entry = match.groupdict()
    print(entry)
    # {'date': '2025-06-14', 'time': '14:30:22', 'level': 'ERROR',
    #  'host': 'web-01', 'message': 'Connection refused to 10.0.2.1:5432'}
```

### Compiled Patterns (Performance)

```python
import re

# For patterns you use repeatedly, compile them once:
LOG_PATTERN = re.compile(
    r"(?P<date>\d{4}-\d{2}-\d{2}) "
    r"(?P<time>\d{2}:\d{2}:\d{2}) "
    r"(?P<level>\w+) "
    r"(?P<host>\S+) "
    r"(?P<message>.*)"
)

# Compiled pattern is faster when used many times:
def parse_log_file(filepath):
    entries = []
    with open(filepath) as f:
        for line in f:
            match = LOG_PATTERN.match(line.strip())
            if match:
                entries.append(match.groupdict())
    return entries

# This is ~2x faster than calling re.search() each time
# because the pattern only gets compiled ONCE.
```

### Parsing Real Log Formats

```python
import re

# NGINX access log:
nginx_line = '10.0.1.50 - - [14/Jun/2025:14:30:22 +0000] "GET /api/health HTTP/1.1" 200 1234 "-" "curl/7.88"'

NGINX_PATTERN = re.compile(
    r'(?P<ip>\S+) \S+ \S+ '
    r'\[(?P<timestamp>[^\]]+)\] '
    r'"(?P<method>\w+) (?P<path>\S+) (?P<protocol>[^"]+)" '
    r'(?P<status>\d+) (?P<size>\d+) '
    r'"(?P<referer>[^"]*)" '
    r'"(?P<user_agent>[^"]*)"'
)

match = NGINX_PATTERN.match(nginx_line)
if match:
    entry = match.groupdict()
    print(f"IP: {entry['ip']}")           # 10.0.1.50
    print(f"Path: {entry['path']}")       # /api/health
    print(f"Status: {entry['status']}")   # 200
    print(f"Size: {entry['size']}")       # 1234


# Syslog format:
syslog_line = "Jun 14 14:30:22 web-01 nginx[1234]: Connection from 10.0.1.50"

SYSLOG_PATTERN = re.compile(
    r"(?P<month>\w+)\s+(?P<day>\d+)\s+(?P<time>\d{2}:\d{2}:\d{2})\s+"
    r"(?P<host>\S+)\s+(?P<service>\w+)\[(?P<pid>\d+)\]:\s+(?P<message>.*)"
)

match = SYSLOG_PATTERN.match(syslog_line)
if match:
    print(match.groupdict())
    # {'month': 'Jun', 'day': '14', 'time': '14:30:22', 'host': 'web-01',
    #  'service': 'nginx', 'pid': '1234', 'message': 'Connection from 10.0.1.50'}


# Key=value parser (common in structured logs):
kv_line = "timestamp=2025-06-14T14:30:22 level=ERROR host=web-01 msg=Connection refused"

kv_pairs = dict(re.findall(r"(\w+)=(\S+)", kv_line))
print(kv_pairs)
# {'timestamp': '2025-06-14T14:30:22', 'level': 'ERROR',
#  'host': 'web-01', 'msg': 'Connection'}
# Note: 'msg' value is truncated at space — we'd need a smarter pattern for multi-word values
```

### DAY 2 EXERCISES

> **EXERCISE 1: Full Log Parser**
>
> Write a function `parse_log(filepath)` that:
> a) Reads a log file line by line
> b) Uses compiled named groups to parse each line
> c) Returns list of dicts with: timestamp, level, host, message
> d) Tracks unparseable lines separately
> e) Test with a 20+ line sample log file you create

> **EXERCISE 2: NGINX Log Analyzer**
>
> Parse NGINX access logs and generate:
> a) Total request count
> b) Count per HTTP status code (200, 404, 500, etc.)
> c) Top 5 most requested paths
> d) Top 5 IP addresses by request count
> e) Average response size
> f) All 5xx error lines

> **EXERCISE 3: Multi-Format Parser**
>
> Write parsers for 3 different log formats:
> a) Standard: "2025-06-14 14:30:22 ERROR web-01 message"
> b) NGINX: full access log format
> c) Key=value: "key1=value1 key2=value2"
> d) Auto-detect format and use the right parser

> **DRILL: 10 Log Parsing Challenges (3 min each)**
>
> 1. Extract all IP addresses from a log line
> 2. Use named groups to parse a timestamp
> 3. Parse "host:port" into separate groups
> 4. Compile a pattern and use it in a loop
> 5. Use groupdict() to get a dict from a match
> 6. Parse a CSV-style line with regex: `re.findall(r'"([^"]*)"', line)`
> 7. Match only ERROR lines: `^.*\bERROR\b.*$` with re.MULTILINE
> 8. Extract numbers from a string: `re.findall(r"-?\d+\.?\d*", text)`
> 9. Replace dates with [DATE]: `re.sub(r"\d{4}-\d{2}-\d{2}", "[DATE]", text)`
> 10. Write a regex for a UUID: `[0-9a-f]{8}-[0-9a-f]{4}-...`

---

## DAY 3: JSON & YAML
**Jun 16, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: json module, yaml parsing, config patterns
> 00:20-01:10 Hands-on: read/write configs and API responses
> 01:10-01:30 Drills: data format challenges

### JSON — The Universal Data Format

```python
import json

# JSON ↔ Python mapping:
# JSON object  {} → Python dict
# JSON array   [] → Python list
# JSON string  "" → Python str
# JSON number     → Python int/float
# JSON true/false → Python True/False
# JSON null       → Python None

# Parse JSON string → Python:
json_string = '{"hostname": "web-01", "port": 8080, "healthy": true}'
data = json.loads(json_string)       # loads = load from String
print(data["hostname"])               # 'web-01'
print(type(data))                     # <class 'dict'>

# Python → JSON string:
server = {"hostname": "web-01", "port": 8080, "tags": ["prod", "us-east"]}
json_string = json.dumps(server, indent=2)    # dumps = dump to String
print(json_string)
# {
#   "hostname": "web-01",
#   "port": 8080,
#   "tags": [
#     "prod",
#     "us-east"
#   ]
# }

# Read JSON file:
with open("config.json") as f:
    config = json.load(f)              # load from File

# Write JSON file:
with open("output.json", "w") as f:
    json.dump(data, f, indent=2)       # dump to File
```

### JSON in SRE Work

```python
import json

# Pattern 1: API response parsing
def parse_api_response(response_text):
    """Parse JSON API response safely."""
    try:
        data = json.loads(response_text)
        return data
    except json.JSONDecodeError as e:
        print(f"Invalid JSON: {e}")
        return None

# Pattern 2: Config file with defaults
def load_config(filepath, defaults=None):
    """Load JSON config with fallback defaults."""
    defaults = defaults or {}
    try:
        with open(filepath) as f:
            config = json.load(f)
        # Merge: config overrides defaults
        return {**defaults, **config}
    except FileNotFoundError:
        print(f"Config not found, using defaults: {filepath}")
        return defaults
    except json.JSONDecodeError as e:
        print(f"Invalid config JSON: {e}")
        return defaults

# Usage:
defaults = {"port": 8080, "log_level": "INFO", "timeout": 30}
config = load_config("app.json", defaults)

# Pattern 3: Pretty-print for debugging
def pp(data):
    """Pretty print any Python data as JSON."""
    print(json.dumps(data, indent=2, default=str))

pp({"server": "web-01", "metrics": {"cpu": 87.5, "memory": 62.0}})
```

### YAML — Human-Friendly Config

```python
# Install: pip install pyyaml
import yaml

# YAML is the preferred format for:
# - Kubernetes manifests
# - Ansible playbooks
# - Docker Compose files
# - Prometheus configs
# - Almost all SRE tool configs

# YAML syntax:
yaml_string = """
server:
  hostname: web-01
  port: 8080
  tags:
    - production
    - us-east
  monitoring:
    enabled: true
    interval: 30
    thresholds:
      cpu: 90
      memory: 85
"""

# Parse YAML → Python:
data = yaml.safe_load(yaml_string)     # ALWAYS use safe_load, never load()!
print(data["server"]["hostname"])       # 'web-01'
print(data["server"]["tags"])           # ['production', 'us-east']
print(data["server"]["monitoring"]["thresholds"]["cpu"])   # 90

# Python → YAML:
output = yaml.dump(data, default_flow_style=False)
print(output)

# Read YAML file:
with open("config.yaml") as f:
    config = yaml.safe_load(f)

# Write YAML file:
with open("output.yaml", "w") as f:
    yaml.dump(data, f, default_flow_style=False)
```

> **WARNING: ALWAYS use yaml.safe_load(), NEVER yaml.load()**
>
> `yaml.load()` can execute arbitrary Python code embedded in YAML.
> This is a SECURITY VULNERABILITY. An attacker could craft a YAML
> file that runs malicious code when parsed.
> `yaml.safe_load()` only parses data, no code execution.

### Multi-Document YAML

```python
# YAML can have multiple documents separated by ---
# Common in Kubernetes manifests

multi_yaml = """
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
"""

# Load all documents:
docs = list(yaml.safe_load_all(multi_yaml))
print(len(docs))              # 2
print(docs[0]["kind"])        # 'Deployment'
print(docs[1]["kind"])        # 'Service'
```

### DAY 3 EXERCISES

> **EXERCISE 1: JSON Config Manager**
>
> Build a ConfigManager class:
> a) `load(filepath)` — read JSON config
> b) `get(key, default)` — safe nested access (support "server.port" dot notation)
> c) `set(key, value)` — set nested values
> d) `save(filepath)` — write back to JSON
> e) `merge(other_config)` — merge two configs (other overrides)
> f) Handle errors: file not found, invalid JSON, missing keys

> **EXERCISE 2: YAML Kubernetes Parser**
>
> Write a tool that reads a multi-document Kubernetes YAML:
> a) Count resources by kind (Deployment, Service, ConfigMap, etc.)
> b) List all container images referenced
> c) Find all resources in a given namespace
> d) Extract all port definitions
> e) Validate that required fields exist (apiVersion, kind, metadata.name)

> **EXERCISE 3: Convert Between Formats**
>
> Write functions:
> a) `json_to_yaml(json_filepath, yaml_filepath)` — convert JSON to YAML
> b) `yaml_to_json(yaml_filepath, json_filepath)` — convert YAML to JSON
> c) Handle multi-document YAML
> d) Preserve comments? (trick question — JSON doesn't support comments)

> **DRILL: 10 JSON/YAML Challenges (2 min each)**
>
> 1. Parse a JSON string with json.loads()
> 2. Write a dict to JSON with indent=2
> 3. Read a JSON file with json.load()
> 4. Parse YAML with yaml.safe_load()
> 5. Write a dict to YAML
> 6. Access a nested value: data["a"]["b"]["c"]
> 7. Safely access with .get(): data.get("key", default)
> 8. Load multi-document YAML with safe_load_all()
> 9. Convert between JSON and YAML
> 10. Why safe_load instead of load?

---

## DAY 4: CSV & DATA PROCESSING
**Jun 17, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: csv module, DictReader, data processing patterns
> 00:20-01:10 Hands-on: process SRE data exports
> 01:10-01:30 Drills: data challenges

### CSV Basics

```python
import csv

# Read CSV with DictReader (recommended — uses headers as keys):
with open("servers.csv") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row)
        # {'hostname': 'web-01', 'ip': '10.0.1.1', 'cpu': '87.5', 'status': 'warning'}
        # Note: ALL values are strings! Convert as needed.

# Read with field access:
with open("servers.csv") as f:
    reader = csv.DictReader(f)
    for row in reader:
        name = row["hostname"]
        cpu = float(row["cpu"])         # convert to number!
        if cpu > 80:
            print(f"WARNING: {name} CPU={cpu}%")

# Write CSV:
servers = [
    {"hostname": "web-01", "ip": "10.0.1.1", "cpu": 87.5},
    {"hostname": "web-02", "ip": "10.0.1.2", "cpu": 45.2},
    {"hostname": "db-01",  "ip": "10.0.2.1", "cpu": 92.0},
]

with open("output.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=["hostname", "ip", "cpu"])
    writer.writeheader()
    writer.writerows(servers)
```

### Processing SRE Data

```python
import csv
from collections import Counter, defaultdict

def analyze_incident_csv(filepath):
    """Analyze an incident report CSV."""
    incidents = []
    with open(filepath) as f:
        reader = csv.DictReader(f)
        for row in reader:
            incidents.append(row)

    # Count by severity:
    severity_counts = Counter(i["severity"] for i in incidents)
    print("Incidents by severity:")
    for sev, count in severity_counts.most_common():
        print(f"  {sev}: {count}")

    # Count by service:
    service_counts = Counter(i["service"] for i in incidents)
    print("\nTop 5 services by incidents:")
    for svc, count in service_counts.most_common(5):
        print(f"  {svc}: {count}")

    # Average resolution time:
    times = [float(i["resolution_minutes"]) for i in incidents
             if i["resolution_minutes"]]
    if times:
        avg = sum(times) / len(times)
        print(f"\nAvg resolution time: {avg:.1f} minutes")

    # Group by team:
    by_team = defaultdict(list)
    for i in incidents:
        by_team[i["team"]].append(i)
    for team, items in sorted(by_team.items()):
        print(f"\n{team}: {len(items)} incidents")

    return incidents
```

### Data Transformation Patterns

```python
import csv
import json

# Pattern 1: CSV → JSON
def csv_to_json(csv_path, json_path):
    with open(csv_path) as f:
        rows = list(csv.DictReader(f))
    with open(json_path, "w") as f:
        json.dump(rows, f, indent=2)

# Pattern 2: Filter and transform
def filter_critical(csv_path, output_path):
    """Extract only critical servers."""
    with open(csv_path) as infile, open(output_path, "w", newline="") as outfile:
        reader = csv.DictReader(infile)
        writer = csv.DictWriter(outfile, fieldnames=reader.fieldnames)
        writer.writeheader()
        for row in reader:
            if float(row.get("cpu", 0)) > 90:
                writer.writerow(row)

# Pattern 3: Aggregate and summarize
def summarize_metrics(csv_path):
    """Compute min/max/avg for each metric column."""
    with open(csv_path) as f:
        rows = list(csv.DictReader(f))

    numeric_cols = ["cpu", "memory", "disk", "latency_ms"]
    for col in numeric_cols:
        values = [float(r[col]) for r in rows if r.get(col)]
        if values:
            print(f"{col}: min={min(values):.1f} max={max(values):.1f} "
                  f"avg={sum(values)/len(values):.1f}")
```

### DAY 4 EXERCISES

> **EXERCISE 1: Server Inventory Report**
>
> Create servers.csv with 20 rows, then write a script that:
> a) Reads the CSV
> b) Groups servers by environment (prod/staging/dev)
> c) Finds servers with CPU > 80% or memory > 85%
> d) Calculates average CPU/memory per environment
> e) Writes a summary report (text file or CSV)

> **EXERCISE 2: Incident Tracker**
>
> Create incidents.csv with columns: id, date, severity, service, team, resolution_minutes
> a) Add 30 sample rows
> b) Find the team with most P1 incidents
> c) Find average resolution time by severity
> d) Find the day with most incidents
> e) Write a "top offenders" report

> **EXERCISE 3: Data Pipeline**
>
> a) Read servers.csv
> b) Transform: add a "status" column based on CPU threshold
> c) Filter: keep only prod servers
> d) Export to JSON (servers.json) AND CSV (filtered_servers.csv)
> e) Read back from JSON to verify

> **DRILL: 10 CSV Challenges (2 min each)**
>
> 1. Read a CSV with DictReader, print first 3 rows
> 2. Write a list of dicts to CSV with DictWriter
> 3. Count rows in a CSV without loading all into memory
> 4. Find the max value of a numeric column
> 5. Filter rows where a column matches a condition
> 6. Add a new computed column to each row
> 7. Sort CSV rows by a column
> 8. Convert CSV to JSON
> 9. Merge two CSV files on a common column
> 10. Handle a CSV with missing values (empty strings)

---

## DAY 5: SUBPROCESS & OS
**Jun 18, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: subprocess.run, capture output, os/shutil
> 00:20-01:10 Hands-on: automate system tasks
> 01:10-01:30 Drills: automation challenges

### subprocess — Run Shell Commands from Python

```python
import subprocess

# Basic: run a command
result = subprocess.run(["ls", "-la", "/var/log"], capture_output=True, text=True)
print(result.stdout)           # command output
print(result.stderr)           # error output
print(result.returncode)       # 0 = success

# IMPORTANT: Use a LIST of args, not a string!
# GOOD: ["ls", "-la", "/var/log"]
# BAD:  "ls -la /var/log" (requires shell=True, which is a security risk)

# Check for errors:
result = subprocess.run(["ls", "/nonexistent"], capture_output=True, text=True)
if result.returncode != 0:
    print(f"Error: {result.stderr}")

# Or use check=True to raise on failure:
try:
    result = subprocess.run(
        ["ls", "/nonexistent"],
        capture_output=True, text=True, check=True
    )
except subprocess.CalledProcessError as e:
    print(f"Command failed (exit {e.returncode}): {e.stderr}")
```

### SRE subprocess Patterns

```python
import subprocess
import json

# Pattern 1: Get system info
def get_hostname():
    result = subprocess.run(["hostname"], capture_output=True, text=True)
    return result.stdout.strip()

def get_uptime():
    result = subprocess.run(["uptime", "-p"], capture_output=True, text=True)
    return result.stdout.strip()

# Pattern 2: Check service status
def is_service_running(service_name):
    result = subprocess.run(
        ["systemctl", "is-active", service_name],
        capture_output=True, text=True
    )
    return result.stdout.strip() == "active"

# Pattern 3: Run with timeout
def run_with_timeout(cmd, timeout_seconds=30):
    try:
        result = subprocess.run(
            cmd, capture_output=True, text=True,
            timeout=timeout_seconds, check=True
        )
        return result.stdout
    except subprocess.TimeoutExpired:
        print(f"Command timed out after {timeout_seconds}s: {cmd}")
        return None
    except subprocess.CalledProcessError as e:
        print(f"Command failed: {e.stderr}")
        return None

# Pattern 4: Parse command output
def get_disk_usage():
    """Parse 'df -h' output into structured data."""
    result = subprocess.run(["df", "-h"], capture_output=True, text=True)
    lines = result.stdout.strip().split("\n")
    header = lines[0].split()
    
    disks = []
    for line in lines[1:]:
        parts = line.split()
        if len(parts) >= 6:
            disks.append({
                "filesystem": parts[0],
                "size": parts[1],
                "used": parts[2],
                "available": parts[3],
                "use_percent": parts[4],
                "mount": parts[5],
            })
    return disks

# Pattern 5: Pipe commands (avoid if possible, but sometimes needed)
def grep_logs(filepath, pattern):
    """Equivalent to: grep pattern filepath."""
    result = subprocess.run(
        ["grep", pattern, filepath],
        capture_output=True, text=True
    )
    return result.stdout.strip().split("\n") if result.stdout else []
```

### os and shutil — File System Operations

```python
import os
import shutil
from pathlib import Path

# os basics:
os.getcwd()                           # current directory
os.listdir("/var/log")                # list directory contents
os.path.exists("/etc/hostname")       # check if path exists
os.path.isfile("/etc/hostname")       # is it a file?
os.path.isdir("/var/log")             # is it a directory?
os.path.getsize("/etc/hostname")      # file size in bytes
os.environ.get("HOME")               # environment variable
os.getenv("PATH", "/usr/bin")         # env var with default

# Create/remove:
os.makedirs("output/reports/2025", exist_ok=True)
os.remove("temp.txt")                 # delete file
os.rmdir("empty_dir")                 # delete empty directory

# shutil — higher-level operations:
shutil.copy("src.txt", "dst.txt")     # copy file
shutil.copy2("src.txt", "dst.txt")    # copy with metadata
shutil.copytree("src_dir", "dst_dir") # copy entire directory
shutil.rmtree("dir_to_delete")        # delete directory tree
shutil.move("old.txt", "new.txt")     # move/rename
shutil.disk_usage("/")                # disk stats

# Disk usage:
usage = shutil.disk_usage("/")
total_gb = usage.total / (1024**3)
used_gb = usage.used / (1024**3)
free_gb = usage.free / (1024**3)
pct = (usage.used / usage.total) * 100
print(f"Disk: {used_gb:.1f}/{total_gb:.1f} GB ({pct:.1f}% used)")

# Prefer pathlib for path operations (Day 4 File 10):
from pathlib import Path
for log in Path("/var/log").glob("*.log"):
    print(f"{log.name}: {log.stat().st_size} bytes")
```

### SRE Automation Script

```python
#!/usr/bin/env python3
"""SRE system health check — combines everything from this week."""

import subprocess
import json
import csv
import re
import os
import logging
from pathlib import Path
from datetime import datetime

logging.basicConfig(level=logging.INFO, format="%(asctime)s [%(levelname)s] %(message)s")
logger = logging.getLogger(__name__)

def collect_system_info():
    """Collect basic system information."""
    info = {}
    
    # Hostname
    result = subprocess.run(["hostname"], capture_output=True, text=True)
    info["hostname"] = result.stdout.strip()
    
    # Uptime
    result = subprocess.run(["uptime", "-s"], capture_output=True, text=True)
    info["up_since"] = result.stdout.strip()
    
    # Disk usage
    usage = os.statvfs("/")
    info["disk_total_gb"] = round((usage.f_blocks * usage.f_frsize) / (1024**3), 1)
    info["disk_free_gb"] = round((usage.f_available * usage.f_frsize) / (1024**3), 1)
    
    return info

def parse_recent_errors(log_path, minutes=60):
    """Parse recent errors from a log file using regex."""
    errors = []
    pattern = re.compile(
        r"(?P<timestamp>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})\s+"
        r"(?P<level>ERROR|CRITICAL)\s+"
        r"(?P<message>.*)"
    )
    
    try:
        with open(log_path) as f:
            for line in f:
                match = pattern.match(line.strip())
                if match:
                    errors.append(match.groupdict())
    except FileNotFoundError:
        logger.warning(f"Log file not found: {log_path}")
    
    return errors

def save_report(info, errors, output_dir="reports"):
    """Save health report as JSON and CSV."""
    Path(output_dir).mkdir(exist_ok=True)
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    
    # JSON report
    report = {"system": info, "errors": errors, "generated": timestamp}
    json_path = f"{output_dir}/health_{timestamp}.json"
    with open(json_path, "w") as f:
        json.dump(report, f, indent=2)
    logger.info(f"JSON report: {json_path}")
    
    # CSV of errors
    if errors:
        csv_path = f"{output_dir}/errors_{timestamp}.csv"
        with open(csv_path, "w", newline="") as f:
            writer = csv.DictWriter(f, fieldnames=["timestamp", "level", "message"])
            writer.writeheader()
            writer.writerows(errors)
        logger.info(f"Error CSV: {csv_path}")

if __name__ == "__main__":
    logger.info("Starting health check")
    info = collect_system_info()
    errors = parse_recent_errors("/var/log/syslog")
    save_report(info, errors)
    logger.info(f"Health check complete: {len(errors)} errors found")
```

### DAY 5 EXERCISES

> **EXERCISE 1: System Info Collector**
>
> Write a script that collects:
> a) Hostname, uptime, kernel version
> b) CPU count, total memory
> c) Disk usage for all mounted filesystems
> d) Top 5 processes by CPU
> e) Output as both JSON and formatted text

> **EXERCISE 2: Log Watcher**
>
> Write a script that:
> a) Takes a log file path and a pattern (regex)
> b) Uses subprocess to run `grep` or reads the file with Python
> c) Parses matching lines with regex
> d) Outputs matches as JSON or CSV (user's choice)
> e) Add a --last-n flag to only check last N lines

> **EXERCISE 3: Deployment Script**
>
> Write a simple deployment automation:
> a) Check if required directories exist (create if not)
> b) Copy files from source to destination
> c) Verify copied files match (compare sizes)
> d) Create a symlink: /opt/app/current → /opt/app/release-v1.0
> e) Log every step
> f) If any step fails, roll back (remove copied files)

> **DRILL: 10 subprocess/os Challenges (2 min each)**
>
> 1. Run `hostname` and capture the output
> 2. Run `df -h` and parse the output
> 3. Check if a command succeeded (returncode == 0)
> 4. Run a command with a 5-second timeout
> 5. Get an environment variable with os.getenv()
> 6. Create a nested directory with os.makedirs()
> 7. Copy a file with shutil.copy2()
> 8. Get disk usage with shutil.disk_usage("/")
> 9. List all .log files with pathlib glob
> 10. Why use subprocess.run(["cmd", "arg"]) instead of "cmd arg"?

---

## WEEK 18 SUMMARY

> **FILE 13 COMPLETE — PYTHON FOR SRE I**
>
> ✓ Regex: patterns, quantifiers, anchors, character classes
> ✓ Regex groups: capture groups, named groups, compile for performance
> ✓ Log parsing: NGINX, syslog, key-value, multi-format
> ✓ JSON: loads/dumps, load/dump, safe parsing, config patterns
> ✓ YAML: safe_load, multi-document, Kubernetes manifests
> ✓ CSV: DictReader/DictWriter, data processing, aggregation
> ✓ subprocess: run commands, capture output, timeout, error handling
> ✓ os/shutil: file operations, disk usage, path management
> ✓ 40+ exercises including full SRE automation script
>
> **INTERVIEW PREP (Apple SRE Python):**
> - "Parse this log file with Python" → regex with named groups + DictReader
> - "How do you run shell commands from Python?" → subprocess.run() with list args
> - "Parse JSON vs YAML?" → json.loads/yaml.safe_load, never yaml.load()
> - "Process a large log file efficiently?" → generators + regex, line by line
> - "Regex for IP address?" → r"\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}"
>
> Next: File 14 — Python for SRE II (Week 19, Jun 21-25)
> Topics: HTTP/API Requests, Concurrency (threading/asyncio), Prometheus Client
