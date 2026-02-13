# PYTHON TRACK — FILE 12 of 48
## PYTHON ADVANCED II
### OOP Deep Dive, Dataclasses, Modules, Packages, Virtual Environments, Logging
**Week 17 | Jun 7 - Jun 11, 2025 | 5 Days | 1.5 hrs/day**

---

> **WHAT THIS WEEK COVERS**
>
> File 09 gave you OOP foundations. File 11 gave you decorators and generators.
> This week COMPLETES your Python toolkit with:
>
> - **OOP deep dive** — abstract classes, protocols, dataclasses
> - **Modules & packages** — organize real projects, not just scripts
> - **Virtual environments** — isolate dependencies (critical for production)
> - **Logging** — the right way to output info (not print!)
>
> After this week, you can write PRODUCTION-QUALITY Python code.

| Day | Topic | Key Skills |
|-----|-------|-----------|
| 1 | OOP Deep Dive | Abstract classes, Protocols, multiple inheritance, mixins |
| 2 | Dataclasses & Named Tuples | @dataclass, field(), __post_init__, NamedTuple |
| 3 | Modules & Packages | import system, __init__.py, relative imports, structure |
| 4 | Virtual Environments & Dependencies | venv, pip, requirements.txt, pip freeze |
| 5 | Logging & Capstone | logging module, handlers, formatters, production patterns |

---

## DAY 1: OOP DEEP DIVE
**Jun 7, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: abstract classes, protocols, mixins
> 00:20-01:10 Hands-on: design patterns for SRE
> 01:10-01:30 Drills: OOP design challenges

### Abstract Base Classes (ABCs)

```python
# An abstract class defines an INTERFACE — "you MUST implement these methods."
# You can't create an instance of an abstract class directly.

from abc import ABC, abstractmethod

class HealthChecker(ABC):
    """All health checkers must implement these methods."""

    @abstractmethod
    def check(self):
        """Run the health check. Must return True/False."""
        pass

    @abstractmethod
    def name(self):
        """Return the name of this check."""
        pass

    def run(self):
        """Concrete method — shared by all subclasses."""
        result = self.check()
        status = "PASS" if result else "FAIL"
        print(f"[{status}] {self.name()}")
        return result

# Can't create abstract class directly:
# checker = HealthChecker()   # TypeError: Can't instantiate abstract class

# Must implement ALL abstract methods:
class DiskCheck(HealthChecker):
    def __init__(self, threshold=90):
        self.threshold = threshold

    def check(self):
        import shutil
        usage = shutil.disk_usage("/")
        pct = (usage.used / usage.total) * 100
        return pct < self.threshold

    def name(self):
        return "Disk Usage"

class PortCheck(HealthChecker):
    def __init__(self, host, port):
        self.host = host
        self.port = port

    def check(self):
        import socket
        try:
            s = socket.socket()
            s.settimeout(3)
            s.connect((self.host, self.port))
            s.close()
            return True
        except (ConnectionError, TimeoutError, OSError):
            return False

    def name(self):
        return f"Port {self.host}:{self.port}"

# Run all checks polymorphically:
checks = [DiskCheck(90), PortCheck("localhost", 22)]
for check in checks:
    check.run()        # each calls its OWN check() and name()
```

> **WHY ABCs MATTER FOR SRE**
>
> When you build a monitoring framework, you want to ensure every
> health check implements `check()` and `name()`. ABCs enforce this
> at class creation time — you get an error immediately if you forget
> a method, not at runtime during a 3 AM incident.

### Protocols (Duck Typing, Formalized)

```python
# Python 3.8+ — a lighter alternative to ABCs
# "If it walks like a duck and quacks like a duck, it's a duck"

from typing import Protocol, runtime_checkable

@runtime_checkable
class Checkable(Protocol):
    """Any object with check() and name() methods."""
    def check(self) -> bool: ...
    def name(self) -> str: ...

# No need to inherit! Just implement the methods:
class CPUCheck:                    # doesn't inherit from anything
    def check(self) -> bool:
        return True                # simplified

    def name(self) -> str:
        return "CPU Usage"

# Works with isinstance:
cpu = CPUCheck()
print(isinstance(cpu, Checkable))   # True — has check() and name()

# Protocols are great when you can't control the base class
# (third-party code) but want to ensure objects have the right methods.
```

### Multiple Inheritance & Mixins

```python
# Python supports multiple inheritance — but use it carefully.

# MIXINS are small, focused classes that add ONE capability.
# They're meant to be combined with other classes.

class JsonMixin:
    """Adds JSON serialization to any class."""
    def to_json(self):
        import json
        return json.dumps(self.__dict__, default=str)

    @classmethod
    def from_json(cls, json_string):
        import json
        data = json.loads(json_string)
        return cls(**data)

class LogMixin:
    """Adds logging capability to any class."""
    def log(self, message, level="INFO"):
        print(f"[{level}] {self.__class__.__name__}: {message}")

class MetricsMixin:
    """Adds basic metrics tracking."""
    def __init_metrics(self):
        if not hasattr(self, '_metrics'):
            self._metrics = {}

    def record_metric(self, name, value):
        self.__init_metrics()
        self._metrics[name] = value

    def get_metrics(self):
        self.__init_metrics()
        return dict(self._metrics)

# Combine mixins with a main class:
class Server(JsonMixin, LogMixin, MetricsMixin):
    def __init__(self, name, ip, port=8080):
        self.name = name
        self.ip = ip
        self.port = port

    def check_health(self):
        self.log(f"Checking health of {self.name}")
        self.record_metric("last_check", "2025-06-07")
        return True

server = Server("web-01", "10.0.1.1")
server.check_health()                         # uses LogMixin
print(server.to_json())                       # uses JsonMixin
print(server.get_metrics())                   # uses MetricsMixin
```

### Method Resolution Order (MRO)

```python
# With multiple inheritance, which method runs when names conflict?
# Python uses C3 linearization (MRO).

class A:
    def greet(self): return "A"

class B(A):
    def greet(self): return "B"

class C(A):
    def greet(self): return "C"

class D(B, C):
    pass

d = D()
print(d.greet())              # 'B' — B comes before C in D's parents

# See the MRO:
print(D.__mro__)
# (D, B, C, A, object) — Python checks in this order

# RULE: left to right in parent list, depth-first, but respects order.
# In practice: put the most specific/important parent first.
```

### DAY 1 EXERCISES

> **EXERCISE 1: Health Check Framework**
>
> Build using ABCs:
> a) Abstract `HealthCheck` with: check(), name(), severity()
> b) `DiskCheck` — checks disk usage
> c) `MemoryCheck` — checks memory usage
> d) `ProcessCheck` — checks if a named process is running
> e) `HealthRunner` that runs all checks and prints a report

> **EXERCISE 2: Mixin Practice**
>
> Create mixins:
> a) `TimestampMixin` — adds created_at, updated_at tracking
> b) `ValidationMixin` — adds validate() that checks required fields
> c) `ComparisonMixin` — adds __eq__, __lt__ based on a key attribute
> d) Combine all three with a Server class

> **EXERCISE 3: Protocol Design**
>
> a) Define a `Restartable` protocol: restart(), status()
> b) Create 3 classes that implement it WITHOUT inheriting
> c) Write a function that accepts any Restartable and restarts it
> d) Verify with isinstance()

> **DRILL: 10 OOP Deep Dive Challenges (3 min each)**
>
> 1. Create an ABC with 2 abstract methods and 1 concrete method
> 2. Try to instantiate the ABC directly — see the error
> 3. Create a child that implements all abstract methods
> 4. Create a mixin that adds a to_dict() method
> 5. Use multiple inheritance to combine 2 mixins with a base class
> 6. Print the MRO of a class with multiple parents
> 7. Create a Protocol and check isinstance without inheritance
> 8. What happens if a child forgets to implement an abstract method?
> 9. Create a mixin that adds __repr__ automatically
> 10. When would you use ABC vs Protocol?

---

## DAY 2: DATACLASSES & NAMED TUPLES
**Jun 8, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: @dataclass, fields, NamedTuple
> 00:20-01:10 Hands-on: replace boilerplate classes
> 01:10-01:30 Drills: dataclass exercises

### The Problem: Boilerplate

```python
# Writing a simple data-holding class is TEDIOUS:

class Server:
    def __init__(self, name, ip, port, environment, cpu=0.0, memory=0.0):
        self.name = name
        self.ip = ip
        self.port = port
        self.environment = environment
        self.cpu = cpu
        self.memory = memory

    def __repr__(self):
        return (f"Server(name={self.name!r}, ip={self.ip!r}, "
                f"port={self.port}, environment={self.environment!r})")

    def __eq__(self, other):
        if not isinstance(other, Server):
            return False
        return (self.name == other.name and self.ip == other.ip
                and self.port == other.port)

# That's 15 lines just to store data + print + compare.
# Dataclasses fix this.
```

### @dataclass — Auto-Generate Boilerplate

```python
from dataclasses import dataclass, field

@dataclass
class Server:
    name: str
    ip: str
    port: int
    environment: str
    cpu: float = 0.0
    memory: float = 0.0

# That's it! Python auto-generates:
# __init__, __repr__, __eq__

# Use it:
web1 = Server("web-01", "10.0.1.1", 8080, "prod")
print(web1)
# Server(name='web-01', ip='10.0.1.1', port=8080, environment='prod', cpu=0.0, memory=0.0)

web1_copy = Server("web-01", "10.0.1.1", 8080, "prod")
print(web1 == web1_copy)      # True (auto __eq__ compares all fields)
```

### Dataclass Options

```python
from dataclasses import dataclass, field
from typing import List

# Frozen (immutable) — like a tuple but with names
@dataclass(frozen=True)
class Config:
    hostname: str
    port: int
    environment: str
    # Config values shouldn't change after creation

config = Config("web-01", 8080, "prod")
# config.port = 9090           # FrozenInstanceError!

# Ordered (sortable) — adds __lt__, __le__, __gt__, __ge__
@dataclass(order=True)
class Metric:
    sort_key: float = field(init=False, repr=False)   # hidden sort field
    name: str
    value: float
    unit: str

    def __post_init__(self):
        self.sort_key = self.value        # sort by value

metrics = [
    Metric("cpu", 87.5, "%"),
    Metric("memory", 62.1, "%"),
    Metric("disk", 45.0, "%"),
]
for m in sorted(metrics):
    print(f"{m.name}: {m.value}{m.unit}")
# disk: 45.0%
# memory: 62.1%
# cpu: 87.5%
```

### field() and __post_init__

```python
from dataclasses import dataclass, field
from typing import List
import time

@dataclass
class Server:
    name: str
    ip: str
    port: int = 8080
    tags: List[str] = field(default_factory=list)    # mutable default!
    created_at: float = field(default_factory=time.time)
    _alerts: List[str] = field(default_factory=list, repr=False)

    def __post_init__(self):
        """Runs AFTER __init__ — use for validation and computed fields."""
        if self.port < 1 or self.port > 65535:
            raise ValueError(f"Invalid port: {self.port}")
        if not self.name:
            raise ValueError("Name is required")

# IMPORTANT: default_factory for mutable defaults!
# This is WRONG (shared list bug):
#   tags: List[str] = []           # ALL instances share the SAME list!
# This is RIGHT:
#   tags: List[str] = field(default_factory=list)   # each gets its own

web1 = Server("web-01", "10.0.1.1")
web1.tags.append("production")
web2 = Server("web-02", "10.0.1.2")
print(web2.tags)                   # [] — separate list, not shared!
```

### NamedTuple — Lightweight Alternative

```python
from typing import NamedTuple

class ServerInfo(NamedTuple):
    name: str
    ip: str
    port: int = 8080

# Behaves like a tuple:
info = ServerInfo("web-01", "10.0.1.1", 8080)
print(info.name)            # 'web-01'
print(info[0])              # 'web-01' (tuple indexing)
name, ip, port = info       # tuple unpacking

# Immutable:
# info.name = "web-02"     # AttributeError

# Can use in sets and as dict keys (hashable):
servers = {info, ServerInfo("web-02", "10.0.1.2")}

# WHEN TO USE WHICH:
# NamedTuple → simple, immutable data (like a record from a database)
# @dataclass → when you need methods, mutation, or complex defaults
# @dataclass(frozen=True) → immutable like NamedTuple but with more features
```

### DAY 2 EXERCISES

> **EXERCISE 1: Convert Classes to Dataclasses**
>
> Take the Server class from File 09 and rewrite as @dataclass:
> a) All the same attributes
> b) Add __post_init__ for validation
> c) Add tags as a list field with default_factory
> d) Add a check_health() method (dataclasses CAN have methods!)
> e) Compare: how many lines saved?

> **EXERCISE 2: Frozen Config**
>
> Create `@dataclass(frozen=True)` for:
> a) DatabaseConfig: host, port, database, username, max_connections
> b) AlertConfig: email_to, slack_channel, pagerduty_key, severity_threshold
> c) Verify they're immutable (try to change a field)
> d) Use them as dict keys (frozen dataclasses are hashable)

> **EXERCISE 3: Sortable Metrics**
>
> Create `@dataclass(order=True)` for Metric:
> a) Fields: name, value, timestamp, unit
> b) Sort by value using sort_key field
> c) Create 10 metrics, sort them, print top 5
> d) Use __post_init__ to validate value is positive

> **EXERCISE 4: NamedTuple Practice**
>
> a) Create `LogEntry = NamedTuple(...)` with timestamp, level, host, message
> b) Parse 10 log lines into LogEntry objects
> c) Group by level using a dict
> d) Compare: NamedTuple vs dataclass for this use case

> **DRILL: 10 Dataclass Challenges (3 min each)**
>
> 1. Create a basic @dataclass with 3 fields
> 2. Add a default value to a field
> 3. Use field(default_factory=list) for a mutable default
> 4. Add __post_init__ validation
> 5. Create a frozen dataclass — try to modify it
> 6. Use @dataclass(order=True) and sort a list of instances
> 7. Add a regular method to a dataclass
> 8. Create a NamedTuple, unpack it
> 9. Convert a dataclass to a dict: `from dataclasses import asdict`
> 10. When would you choose NamedTuple over dataclass?

---

## DAY 3: MODULES & PACKAGES
**Jun 9, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: import system, creating modules/packages
> 00:20-01:10 Hands-on: structure a real project
> 01:10-01:30 Drills: import challenges

### Modules — A Single .py File

```python
# A module is just a .py file. When you import it, Python runs the file
# and makes its contents available.

# File: server_utils.py
"""Utility functions for server management."""

DEFAULT_PORT = 8080

def check_health(hostname):
    """Check if a server is healthy."""
    return True

def parse_config(filepath):
    """Parse a config file."""
    return {"port": DEFAULT_PORT}

class Server:
    def __init__(self, name, ip):
        self.name = name
        self.ip = ip

# File: main.py
import server_utils

server_utils.check_health("web-01")
print(server_utils.DEFAULT_PORT)
s = server_utils.Server("web-01", "10.0.1.1")

# Or import specific items:
from server_utils import check_health, Server, DEFAULT_PORT
check_health("web-01")

# Or import with alias:
import server_utils as su
su.check_health("web-01")

# Import everything (avoid this — pollutes namespace):
from server_utils import *
```

### Packages — Directories of Modules

```
A package is a DIRECTORY containing modules and an __init__.py file.

my_project/
├── main.py
├── monitoring/                  ← package (directory)
│   ├── __init__.py              ← makes it a package (can be empty)
│   ├── checks.py                ← module
│   ├── alerts.py                ← module
│   └── exporters/               ← sub-package
│       ├── __init__.py
│       ├── prometheus.py
│       └── datadog.py
├── config/                      ← another package
│   ├── __init__.py
│   ├── loader.py
│   └── validator.py
└── utils/
    ├── __init__.py
    └── helpers.py
```

```python
# Importing from packages:
from monitoring.checks import DiskCheck, PortCheck
from monitoring.alerts import send_alert
from monitoring.exporters.prometheus import PrometheusExporter
from config.loader import load_config

# __init__.py can re-export for convenience:
# monitoring/__init__.py:
from .checks import DiskCheck, PortCheck
from .alerts import send_alert

# Now users can do:
from monitoring import DiskCheck, send_alert
# Instead of the longer path
```

### __init__.py — Package Initialization

```python
# monitoring/__init__.py

"""Monitoring package for SRE tools."""

# Re-export commonly used items:
from .checks import DiskCheck, PortCheck, CPUCheck
from .alerts import send_alert, AlertManager

# Package-level constants:
VERSION = "1.0.0"
DEFAULT_INTERVAL = 60

# Package-level setup:
import logging
logger = logging.getLogger(__name__)

# Control what 'from monitoring import *' exposes:
__all__ = ["DiskCheck", "PortCheck", "CPUCheck", "send_alert", "AlertManager"]
```

### Relative vs Absolute Imports

```python
# INSIDE a package, you have two options:

# Absolute import (full path from project root):
from monitoring.checks import DiskCheck
from config.loader import load_config

# Relative import (relative to current file):
# . = current package
# .. = parent package
from .checks import DiskCheck          # same package
from .alerts import send_alert         # same package
from ..config.loader import load_config  # parent's sibling package

# RULE: Use absolute imports for clarity.
# Use relative imports only within a package's internal files.
```

### The if __name__ == "__main__" Pattern

```python
# server_utils.py

def check_health(hostname):
    return True

def main():
    """Run when this file is executed directly."""
    print("Running health checks...")
    result = check_health("web-01")
    print(f"Result: {result}")

# This block ONLY runs when you execute: python3 server_utils.py
# It does NOT run when you import: import server_utils
if __name__ == "__main__":
    main()

# WHY?
# When Python runs a file directly: __name__ = "__main__"
# When Python imports a file: __name__ = "server_utils"
# This lets a file be BOTH a module (importable) and a script (runnable).
```

### Real Project Structure

```
sre-toolkit/
├── pyproject.toml              ← modern project config (replaces setup.py)
├── README.md
├── requirements.txt            ← dependencies
├── src/
│   └── sre_toolkit/            ← main package
│       ├── __init__.py
│       ├── monitoring/
│       │   ├── __init__.py
│       │   ├── checks.py
│       │   └── alerts.py
│       ├── config/
│       │   ├── __init__.py
│       │   └── loader.py
│       └── utils/
│           ├── __init__.py
│           └── helpers.py
├── tests/                      ← test files
│   ├── test_checks.py
│   ├── test_alerts.py
│   └── test_loader.py
└── scripts/                    ← CLI scripts
    └── run_checks.py
```

### DAY 3 EXERCISES

> **EXERCISE 1: Create a Module**
>
> a) Create `server_utils.py` with 3 functions and 1 class
> b) Create `main.py` that imports and uses them
> c) Test all 3 import styles: `import`, `from...import`, `as`
> d) Add `if __name__ == "__main__"` to server_utils.py

> **EXERCISE 2: Create a Package**
>
> Build this structure:
> ```
> monitoring/
> ├── __init__.py
> ├── checks.py      (DiskCheck, CPUCheck, PortCheck)
> ├── alerts.py      (Alert class, send_alert function)
> └── formatters.py  (format_report, format_table)
> ```
> a) Create all files with real code
> b) Use __init__.py to re-export key items
> c) Import from a main.py outside the package
> d) Test relative imports between modules in the package

> **EXERCISE 3: Project Layout**
>
> Create the full `sre-toolkit/` structure shown above:
> a) Main package with sub-packages
> b) At least 2 classes and 3 functions across modules
> c) Tests directory with at least 2 test files
> d) requirements.txt (even if empty for now)
> e) README.md with brief description

> **DRILL: 10 Module/Package Challenges (3 min each)**
>
> 1. Create a .py file, import it from another file
> 2. Use `from module import specific_thing`
> 3. Use `import module as alias`
> 4. Create a directory with __init__.py — import from it
> 5. What does __all__ control?
> 6. What's the difference between `import x` and `from x import y`?
> 7. Add `if __name__ == "__main__"` — verify it only runs when executed
> 8. Create a relative import inside a package
> 9. What happens if __init__.py is missing?
> 10. Run `python3 -c "import sys; print(sys.path)"` — what do you see?

---

## DAY 4: VIRTUAL ENVIRONMENTS & DEPENDENCIES
**Jun 10, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: venv, pip, requirements.txt, dependency management
> 00:20-01:10 Hands-on: set up a real project environment
> 01:10-01:30 Drills: dependency challenges

### Why Virtual Environments?

```
PROBLEM: You have two projects:
  Project A needs requests==2.28
  Project B needs requests==2.31

Without venvs, both share the SAME Python packages.
Installing 2.31 breaks Project A. Installing 2.28 breaks Project B.

SOLUTION: Virtual environments give each project its OWN
isolated set of packages. Project A and B each have their
own copy of requests at the version they need.

This is CRITICAL for production SRE:
  - Your monitoring script needs library version X
  - Your deployment tool needs version Y
  - Without isolation, upgrading one breaks the other
```

### Creating and Using venvs

```bash
# Create a virtual environment:
python3 -m venv myproject-env

# What this creates:
# myproject-env/
# ├── bin/          ← scripts (activate, python, pip)
# │   ├── activate
# │   ├── python → python3.10
# │   └── pip
# ├── lib/          ← installed packages go here
# └── pyvenv.cfg

# Activate the venv:
source myproject-env/bin/activate

# Your prompt changes:
# (myproject-env) $

# Now python and pip use the venv's versions:
which python           # /path/to/myproject-env/bin/python
which pip              # /path/to/myproject-env/bin/pip

# Install packages (isolated!):
pip install requests flask prometheus-client

# Deactivate when done:
deactivate
```

### Managing Dependencies

```bash
# See what's installed:
pip list
pip show requests          # detailed info about one package

# Freeze current packages (exact versions):
pip freeze > requirements.txt

# Example requirements.txt:
# certifi==2023.7.22
# charset-normalizer==3.3.2
# Flask==3.0.0
# prometheus-client==0.19.0
# requests==2.31.0

# Install from requirements.txt (reproduces exact environment):
pip install -r requirements.txt

# This is how you share your environment:
# 1. You develop and pip freeze > requirements.txt
# 2. Teammate clones repo
# 3. Teammate runs: python3 -m venv env && source env/bin/activate
# 4. Teammate runs: pip install -r requirements.txt
# 5. Exact same packages!
```

### requirements.txt Best Practices

```bash
# Pin exact versions for production (reproducible):
# requirements.txt
requests==2.31.0
flask==3.0.0
prometheus-client==0.19.0
pyyaml==6.0.1

# For development, you might allow ranges:
# requirements-dev.txt
pytest>=7.0
black>=23.0
mypy>=1.0
flake8>=6.0

# Install both:
pip install -r requirements.txt
pip install -r requirements-dev.txt

# Upgrade a specific package:
pip install --upgrade requests

# Upgrade all packages (careful!):
pip install --upgrade -r requirements.txt
```

### Project Setup Script

```bash
#!/bin/bash
# setup.sh — set up a new SRE project

PROJECT_NAME=${1:-"sre-project"}

# Create project structure
mkdir -p $PROJECT_NAME/{src,tests,scripts,config}
cd $PROJECT_NAME

# Create venv
python3 -m venv .venv
source .venv/bin/activate

# Create requirements files
cat > requirements.txt << 'EOF'
requests==2.31.0
pyyaml==6.0.1
prometheus-client==0.19.0
click==8.1.7
EOF

cat > requirements-dev.txt << 'EOF'
pytest>=7.0
black>=23.0
mypy>=1.0
EOF

# Install
pip install -r requirements.txt
pip install -r requirements-dev.txt

# Create .gitignore
cat > .gitignore << 'EOF'
.venv/
__pycache__/
*.pyc
.mypy_cache/
.pytest_cache/
dist/
*.egg-info/
EOF

echo "Project $PROJECT_NAME created and ready!"
```

### DAY 4 EXERCISES

> **EXERCISE 1: Set Up a Project**
>
> a) Create a new directory for an SRE monitoring tool
> b) Create a virtual environment inside it
> c) Activate it, verify python and pip point to venv
> d) Install: requests, pyyaml, prometheus-client
> e) Freeze to requirements.txt
> f) Deactivate, delete venv, recreate from requirements.txt

> **EXERCISE 2: Dependency Conflict**
>
> a) Create venv A, install requests==2.28.0
> b) Create venv B, install requests==2.31.0
> c) Verify they're independent (pip show requests in each)
> d) This demonstrates why venvs are essential

> **EXERCISE 3: Project Template**
>
> Create a reusable project template:
> a) setup.sh script that creates structure + venv + installs
> b) .gitignore for Python projects
> c) requirements.txt and requirements-dev.txt
> d) README.md with setup instructions
> e) Run it and verify everything works

> **DRILL: 10 venv/pip Challenges (2 min each)**
>
> 1. Create a venv, activate it
> 2. Install a package, verify with pip list
> 3. Freeze to requirements.txt
> 4. pip show a package — what info does it give?
> 5. Deactivate the venv
> 6. Delete and recreate from requirements.txt
> 7. What's inside the venv directory?
> 8. Why should .venv be in .gitignore?
> 9. What's the difference between pip install X and pip install X==1.0?
> 10. How do you upgrade a single package?

---

## DAY 5: LOGGING & CAPSTONE
**Jun 11, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: logging module, handlers, formatters
> 00:20-01:10 Hands-on: capstone project
> 01:10-01:30 Drills: logging + review

### Why Logging, Not print()

```python
# print() is for DEVELOPMENT. logging is for PRODUCTION.

# print() problems:
# 1. No severity levels (is this info? warning? error?)
# 2. No timestamps
# 3. Can't easily write to files AND console
# 4. Can't filter by level
# 5. No context (which module? which function?)
# 6. Hard to turn off in production

# logging solves ALL of these.
```

### logging Basics

```python
import logging

# Basic setup (quick and dirty):
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)

# Create a logger:
logger = logging.getLogger(__name__)

# Log at different levels:
logger.debug("Detailed debugging info")        # level 10
logger.info("Normal operation")                 # level 20
logger.warning("Something unexpected")          # level 30
logger.error("Something failed")                # level 40
logger.critical("System is down!")              # level 50

# Output (only INFO and above, since we set level=INFO):
# 2025-06-11 14:30:01 [INFO] __main__: Normal operation
# 2025-06-11 14:30:01 [WARNING] __main__: Something unexpected
# 2025-06-11 14:30:01 [ERROR] __main__: Something failed
# 2025-06-11 14:30:01 [CRITICAL] __main__: System is down!
```

### Production Logging Setup

```python
import logging
import logging.handlers
import sys

def setup_logging(name, log_file=None, level=logging.INFO):
    """Configure production-quality logging."""
    logger = logging.getLogger(name)
    logger.setLevel(level)

    # Format:
    formatter = logging.Formatter(
        fmt="%(asctime)s [%(levelname)-8s] %(name)s:%(lineno)d - %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S"
    )

    # Console handler (stderr):
    console = logging.StreamHandler(sys.stderr)
    console.setLevel(logging.INFO)
    console.setFormatter(formatter)
    logger.addHandler(console)

    # File handler (with rotation):
    if log_file:
        file_handler = logging.handlers.RotatingFileHandler(
            log_file,
            maxBytes=10 * 1024 * 1024,     # 10 MB per file
            backupCount=5,                  # keep 5 rotated files
        )
        file_handler.setLevel(logging.DEBUG)    # log everything to file
        file_handler.setFormatter(formatter)
        logger.addHandler(file_handler)

    return logger

# Usage:
logger = setup_logging("sre-toolkit", log_file="app.log")

logger.info("Application started")
logger.warning("High CPU detected on web-01: 87.5%")
logger.error("Connection failed to db-01:5432", exc_info=True)
```

### Logging with Context

```python
import logging

logger = logging.getLogger(__name__)

# Log with variables (use % style or f-string):
hostname = "web-01"
cpu = 87.5
logger.warning("High CPU on %s: %.1f%%", hostname, cpu)
# Or:
logger.warning(f"High CPU on {hostname}: {cpu:.1f}%")

# Log exceptions with traceback:
try:
    result = 10 / 0
except ZeroDivisionError:
    logger.error("Calculation failed", exc_info=True)
    # This prints the FULL traceback to the log!

# Or use logger.exception() (shortcut for error + exc_info):
try:
    result = 10 / 0
except ZeroDivisionError:
    logger.exception("Calculation failed")

# Structured logging with extra data:
logger.info(
    "Health check completed",
    extra={"host": "web-01", "status": "healthy", "duration_ms": 45},
)
```

### SRE Logging Patterns

```python
# Pattern 1: Operation timer
import time

def timed_operation(operation_name):
    """Log start, duration, and result of an operation."""
    start = time.time()
    logger.info(f"Starting: {operation_name}")
    try:
        yield
        elapsed = time.time() - start
        logger.info(f"Completed: {operation_name} ({elapsed:.2f}s)")
    except Exception as e:
        elapsed = time.time() - start
        logger.error(f"Failed: {operation_name} ({elapsed:.2f}s): {e}")
        raise

# Pattern 2: Per-module loggers
# monitoring/checks.py:
logger = logging.getLogger(__name__)    # logger name = "monitoring.checks"

# monitoring/alerts.py:
logger = logging.getLogger(__name__)    # logger name = "monitoring.alerts"

# Control verbosity per module:
logging.getLogger("monitoring.checks").setLevel(logging.DEBUG)
logging.getLogger("monitoring.alerts").setLevel(logging.WARNING)

# Pattern 3: Never log sensitive data
# BAD:
logger.info(f"Connecting with password={password}")
# GOOD:
logger.info(f"Connecting to {host} as {username}")
```

### CAPSTONE: Production SRE Tool

> **Build a complete, production-quality SRE monitoring tool:**
>
> **Project structure:**
> ```
> sre-monitor/
> ├── .venv/
> ├── requirements.txt
> ├── src/
> │   └── sre_monitor/
> │       ├── __init__.py
> │       ├── models.py         (dataclasses: Server, Metric, Alert)
> │       ├── checks.py         (ABC HealthCheck + implementations)
> │       ├── alerts.py         (AlertManager with logging)
> │       └── runner.py         (main orchestration)
> ├── tests/
> │   ├── test_models.py
> │   ├── test_checks.py
> │   └── test_alerts.py
> └── scripts/
>     └── run_monitor.py
> ```
>
> **Requirements:**
> - Use @dataclass for Server, Metric, Alert
> - Use ABC for HealthCheck base class
> - Use logging (not print) everywhere
> - Use virtual environment
> - Use proper package structure with __init__.py
> - Include at least 5 tests
> - Include requirements.txt

### DAY 5 EXERCISES

> **EXERCISE 1: Logging Setup**
>
> a) Set up logging with both console and file handlers
> b) Use RotatingFileHandler with 5MB max
> c) Log at all 5 levels, verify output
> d) Log an exception with full traceback
> e) Create per-module loggers for 2 different modules

> **EXERCISE 2: Replace print with logging**
>
> Take any script you wrote in Files 07-11:
> a) Replace ALL print() calls with appropriate logger calls
> b) Choose the right level for each (debug/info/warning/error)
> c) Add timestamps and module names
> d) Write to both console and file

> **EXERCISE 3: Build the Capstone**
>
> Follow the project structure above and build the full tool.
> This is your first REAL Python project.

> **DRILL: 10 Logging/Review Challenges (3 min each)**
>
> 1. Create a logger, log at INFO level
> 2. Set up basicConfig with a custom format
> 3. Log an exception with exc_info=True
> 4. Create a RotatingFileHandler
> 5. Use per-module loggers (__name__)
> 6. Set different log levels for different modules
> 7. What's the difference between logger.error and logger.exception?
> 8. Create a @dataclass with 5 fields and __post_init__ validation
> 9. Create a package with __init__.py that re-exports key items
> 10. Set up a venv, install packages, freeze requirements

---

## WEEK 17 SUMMARY

> **FILE 12 COMPLETE — PYTHON ADVANCED II**
>
> ✓ ABCs: abstract classes, enforced interfaces, @abstractmethod
> ✓ Protocols: structural typing, duck typing formalized
> ✓ Mixins: multiple inheritance done right, add focused capabilities
> ✓ Dataclasses: @dataclass, field(), frozen, ordered, __post_init__
> ✓ NamedTuples: lightweight immutable records
> ✓ Modules: import system, __name__, __all__
> ✓ Packages: __init__.py, directory structure, relative imports
> ✓ Virtual environments: venv, pip, requirements.txt
> ✓ Logging: levels, handlers, formatters, production patterns
> ✓ 40+ exercises including full project capstone
>
> **INTERVIEW PREP:**
> - "What is an abstract class?" → Defines an interface with @abstractmethod.
>   Can't instantiate directly. Children MUST implement abstract methods.
> - "What is a dataclass?" → Decorator that auto-generates __init__, __repr__,
>   __eq__. Reduces boilerplate for data-holding classes.
> - "What is a virtual environment?" → Isolated Python package installation.
>   Each project gets its own dependencies. Created with python3 -m venv.
> - "Why use logging instead of print?" → Levels, timestamps, file output,
>   filtering, rotation, per-module control. print is for dev only.
> - "What is __init__.py?" → Makes a directory a Python package.
>   Can re-export items and run package initialization code.
>
> Next: File 13 — Python for SRE I (Week 18, Jun 14-18)
> Topics: Regex, JSON/YAML Parsing, CSV, subprocess, Logging (applied)
