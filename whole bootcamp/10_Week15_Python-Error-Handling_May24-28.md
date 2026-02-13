# PYTHON TRACK — FILE 10 of 48
## PYTHON ERROR HANDLING & FILE I/O
### Exceptions, try/except/finally, Custom Exceptions, File Operations, Context Managers, pytest
**Week 15 | May 24 - May 28, 2025 | 5 Days | 1.5 hrs/day**

---

> **WHY ERROR HANDLING IS AN SRE SUPERPOWER**
>
> In production, EVERYTHING fails:
> - Network calls timeout
> - Files don't exist
> - Disk fills up
> - Config has bad values
> - APIs return unexpected data
>
> The difference between a script that crashes at 3 AM and one that
> handles the failure, logs it, retries, and pages you? ERROR HANDLING.
>
> This week you'll learn to write code that EXPECTS failure and
> handles it gracefully — the #1 quality of production-grade SRE code.

| Day | Topic | Key Skills |
|-----|-------|-----------|
| 1 | Exceptions & try/except | What exceptions are, catching them, common types |
| 2 | Advanced Exception Handling | Multiple excepts, else, finally, raising, chaining |
| 3 | Custom Exceptions & Patterns | Build your own, SRE error patterns, retry logic |
| 4 | File I/O & Context Managers | Read/write files, with statement, pathlib |
| 5 | Testing with pytest & Capstone | Write tests, test exceptions, full project |

---

## DAY 1: EXCEPTIONS & try/except
**May 24, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: what exceptions are, try/except basics
> 00:20-01:10 Hands-on: catch and handle errors
> 01:10-01:30 Drills: exception exercises

### What Is an Exception?

```python
# An exception is Python's way of saying "something went wrong."
# If you don't handle it, your program CRASHES.

# Try each of these one at a time. See the error message.

# 1. ZeroDivisionError:
result = 10 / 0

# 2. FileNotFoundError:
f = open("nonexistent.txt")

# 3. KeyError:
config = {"port": 8080}
value = config["hostname"]

# 4. IndexError:
servers = ["web-01", "web-02"]
print(servers[5])

# 5. TypeError:
result = "hello" + 42

# 6. ValueError:
number = int("hello")

# 7. AttributeError:
name = "hello"
name.append("!")
```

> **READ THE TRACEBACK — IT'S YOUR FRIEND**
>
> When Python crashes, it prints a traceback. Read it BOTTOM UP:
> ```
> Traceback (most recent call last):
>   File "script.py", line 5, in <module>      ← where it happened
>     result = 10 / 0                           ← the line that crashed
> ZeroDivisionError: division by zero            ← WHAT went wrong
> ```
> The LAST line is the most important — the exception type and message.

### The try/except Block

```python
# try/except lets you CATCH exceptions and handle them
# instead of crashing.

# Basic structure:
try:
    result = 10 / 0
except ZeroDivisionError:
    print("Cannot divide by zero!")
    result = 0

print(f"Result: {result}")         # Result: 0 (didn't crash!)

# With the exception object (use 'as e'):
try:
    number = int("hello")
except ValueError as e:
    print(f"Conversion error: {e}")    # Conversion error: invalid literal...
    number = 0
```

### Catching Specific Exceptions

```python
# ALWAYS catch SPECIFIC exceptions. Here's why:

# BAD — catches EVERYTHING (hides bugs):
try:
    value = config["hostname"]
except:                            # bare except — NEVER DO THIS
    print("Something went wrong")
# You had a typo in variable name? You'd never know.

# GOOD — specific exception:
try:
    value = config["hostname"]
except KeyError:
    print("Key 'hostname' not found in config")
    value = "unknown"

# GOOD — with the exception details:
try:
    value = config["hostname"]
except KeyError as e:
    print(f"Missing key: {e}")     # Missing key: 'hostname'
    value = "unknown"
```

### Multiple except Blocks

```python
def parse_server_config(config_line):
    """Parse 'hostname:port' into (hostname, port)."""
    try:
        parts = config_line.split(":")
        hostname = parts[0]
        port = int(parts[1])
        return hostname, port
    except IndexError:
        print(f"Bad format (missing port): '{config_line}'")
        return None, None
    except ValueError:
        print(f"Port is not a number: '{config_line}'")
        return None, None
    except AttributeError:
        print(f"Input is not a string: {config_line}")
        return None, None

# Test all cases:
print(parse_server_config("web-01:8080"))    # ('web-01', 8080)
print(parse_server_config("web-01"))          # Bad format
print(parse_server_config("web-01:abc"))      # Port not a number
print(parse_server_config(12345))             # Not a string

# Catch multiple in one block:
try:
    result = some_operation()
except (ValueError, TypeError, KeyError) as e:
    print(f"Data error: {type(e).__name__}: {e}")
```

### Common SRE Patterns

```python
# Pattern 1: Safe type conversion
user_input = "not_a_number"
try:
    port = int(user_input)
except ValueError:
    print(f"Invalid port: '{user_input}', using default")
    port = 8080

# Pattern 2: Safe file reading
try:
    with open("/etc/hostname") as f:
        hostname = f.read().strip()
except FileNotFoundError:
    hostname = "unknown"
except PermissionError:
    hostname = "access-denied"

# Pattern 3: Safe dict access (try/except vs .get)
# For simple key access, .get() is better:
value = config.get("timeout", 30)

# But try/except is needed for complex operations:
try:
    port = int(config["port"])
    if port < 1 or port > 65535:
        raise ValueError(f"Bad port: {port}")
except (KeyError, ValueError) as e:
    print(f"Config error: {e}")
    port = 8080
```

### DAY 1 EXERCISES

> **EXERCISE 1: Safe Calculator**
>
> Write a function `safe_divide(a, b)` that:
> a) Returns a / b
> b) Handles ZeroDivisionError (return 0)
> c) Handles TypeError when a or b isn't a number (return None)
> d) Test with: (10, 2), (10, 0), ("hello", 2), (10, "world")

> **EXERCISE 2: Config Parser**
>
> Write a function `parse_config(lines)` that takes a list of "key=value" strings:
> a) Returns a dictionary of key-value pairs
> b) Skips blank lines without crashing
> c) Handles lines without "=" (log warning, skip)
> d) Converts values to int if possible, else keep as string
> e) Test with: `["port=8080", "host=localhost", "debug=true", "badline", "", "timeout=30"]`

> **EXERCISE 3: Server Health Check Simulator**
>
> Write a function that takes a server dict and checks:
> a) "name" key exists (KeyError if missing)
> b) "cpu" is a valid number (ValueError/TypeError)
> c) "cpu" is between 0-100 (ValueError if not)
> d) Catch each error separately, print a clear message
> e) Return "healthy"/"warning"/"critical"/"error"

> **DRILL: 10 Exception Challenges (2 min each)**
>
> 1. Catch a ZeroDivisionError, print a friendly message
> 2. Catch a FileNotFoundError when opening a fake file
> 3. Catch a ValueError when converting "abc" to int
> 4. Catch a KeyError when accessing a missing dict key
> 5. Use `as e` to print the actual error message
> 6. Catch two different exceptions in one except block
> 7. What happens if you catch Exception? Test it.
> 8. What happens if an exception is NOT caught? (read the traceback)
> 9. Wrap a list index access in try/except IndexError
> 10. Print `type(e).__name__` to get the exception class name

---

## DAY 2: ADVANCED EXCEPTION HANDLING
**May 25, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: else, finally, raising, exception chaining
> 00:20-01:10 Hands-on: advanced error handling
> 01:10-01:30 Drills: production patterns

### The Full try/except/else/finally

```python
try:
    result = int(input("Enter a number: "))
except ValueError:
    print("That's not a valid number!")
    result = 0
else:
    # Runs ONLY if NO exception occurred
    print(f"Great! You entered {result}")
finally:
    # ALWAYS runs — even if there's a return or crash
    print("Input processing complete.")

# When does each run?
# No error:    try → else → finally
# Error caught: try (partial) → except → finally
# Error NOT caught: try (partial) → finally → CRASH
```

### Raising Exceptions

```python
# YOUR code should raise exceptions to signal errors to callers.

def set_cpu(server_name, value):
    """Set CPU usage. Must be 0-100."""
    if not isinstance(value, (int, float)):
        raise TypeError(f"CPU must be a number, got {type(value).__name__}")
    if value < 0 or value > 100:
        raise ValueError(f"CPU must be 0-100, got {value}")
    print(f"{server_name}: CPU set to {value}%")

# Normal:
set_cpu("web-01", 85.5)

# Catching our raised exceptions:
try:
    set_cpu("web-01", 150)
except ValueError as e:
    print(f"Error: {e}")         # Error: CPU must be 0-100, got 150

try:
    set_cpu("web-01", "high")
except TypeError as e:
    print(f"Error: {e}")         # Error: CPU must be a number, got str
```

### Re-raising and Chaining

```python
# Re-raise: log the error but let it propagate up
def process_config(filepath):
    try:
        with open(filepath) as f:
            return parse(f.read())
    except FileNotFoundError:
        print(f"CRITICAL: Config missing: {filepath}")
        raise                    # re-raises the SAME exception

# Chain: add context to the original error
    except FileNotFoundError as e:
        raise RuntimeError(f"Cannot start: {filepath}") from e
        # 'from e' preserves the original traceback
```

### Exception Hierarchy

```
BaseException
  ├── KeyboardInterrupt     ← Ctrl+C (DON'T catch normally)
  ├── SystemExit            ← sys.exit() (DON'T catch)
  └── Exception             ← catch this or its children
        ├── ValueError
        ├── TypeError
        ├── KeyError
        ├── IndexError
        ├── FileNotFoundError
        ├── PermissionError
        ├── ConnectionError
        │     ├── ConnectionRefusedError
        │     └── ConnectionResetError
        ├── TimeoutError
        └── RuntimeError

RULE: Catch the most SPECIFIC exception you can.
NEVER catch BaseException — it catches Ctrl+C too!
```

### SRE Pattern: Graceful Degradation

```python
def get_server_metrics(hostname):
    """Try multiple sources, degrade gracefully."""
    
    # Try primary:
    try:
        return query_prometheus(hostname)
    except ConnectionError:
        print("Prometheus unavailable, trying fallback...")
    
    # Try fallback:
    try:
        return query_local_agent(hostname)
    except (ConnectionError, TimeoutError):
        print("Local agent unavailable, using cache...")
    
    # Last resort:
    try:
        metrics = read_cache(hostname)
        metrics["cached"] = True
        return metrics
    except FileNotFoundError:
        return {"hostname": hostname, "error": "no data"}

# This function NEVER crashes. Always returns something.
# This is the SRE mindset.
```

### DAY 2 EXERCISES

> **EXERCISE 1: try/except/else/finally**
>
> Write a function `safe_read_number(prompt)`:
> a) Uses input() to ask for a number
> b) except: print error message if not valid
> c) else: print "Valid number: {n}" only on success
> d) finally: print "Input attempt complete"
> e) Return number on success, None on failure

> **EXERCISE 2: Raise Your Own**
>
> Write a class `ServerConfig`:
> a) __init__ takes name, port, environment
> b) Raise ValueError if name is empty
> c) Raise ValueError if port not 1-65535
> d) Raise ValueError if env not in ("prod", "staging", "dev")
> e) Write tests that create bad configs and catch errors

> **EXERCISE 3: Graceful Degradation**
>
> Write `get_hostname()` that tries 3 methods:
> a) Try reading /etc/hostname
> b) Try os.popen("hostname").read()
> c) Fall back to "unknown-host"
> d) NEVER crash, always return a string

> **DRILL: 10 Advanced Exception Challenges (3 min each)**
>
> 1. Write a function that raises ValueError if input is negative
> 2. Use `raise ... from e` to chain two exceptions
> 3. Verify else only runs on success
> 4. Verify finally runs even after an error
> 5. Write try/except for ConnectionError
> 6. Re-raise an exception with bare `raise`
> 7. Use `traceback.print_exc()` to print traceback without crashing
> 8. What happens if you raise inside an except block?
> 9. Write a function that NEVER crashes
> 10. Compare: exceptions vs return codes

---

## DAY 3: CUSTOM EXCEPTIONS & SRE PATTERNS
**May 26, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: custom exceptions, retry logic, error hierarchies
> 00:20-01:10 Hands-on: production-quality error handling
> 01:10-01:30 Drills: SRE patterns

### Custom Exceptions

```python
# Custom exceptions = clearer code + better error handling

class ServerError(Exception):
    """Base for server errors."""
    pass

class ServerNotFoundError(ServerError):
    """Server hostname can't be resolved."""
    pass

class ServerUnhealthyError(ServerError):
    """Server fails health checks."""
    pass

class ConfigError(Exception):
    """Base for config errors."""
    pass

class MissingConfigError(ConfigError):
    """Required config key missing."""
    def __init__(self, key):
        self.key = key
        super().__init__(f"Required config key missing: '{key}'")

class InvalidConfigError(ConfigError):
    """Config value is invalid."""
    def __init__(self, key, value, reason):
        self.key = key
        self.value = value
        super().__init__(f"Invalid config '{key}={value}': {reason}")
```

Using them:

```python
def load_config(filepath):
    """Load and validate configuration."""
    required = ["hostname", "port", "environment"]
    
    try:
        with open(filepath) as f:
            config = {}
            for line in f:
                line = line.strip()
                if not line or line.startswith("#"):
                    continue
                key, _, value = line.partition("=")
                config[key.strip()] = value.strip()
    except FileNotFoundError:
        raise ConfigError(f"Config file not found: {filepath}")
    
    for key in required:
        if key not in config:
            raise MissingConfigError(key)
    
    try:
        port = int(config["port"])
    except ValueError:
        raise InvalidConfigError("port", config["port"], "must be a number")
    
    return config

# Catch at the right level:
try:
    config = load_config("server.conf")
except MissingConfigError as e:
    print(f"Missing: {e.key}")
except ConfigError as e:           # catches ALL config errors
    print(f"Config problem: {e}")
```

### Retry with Exponential Backoff

```python
import time
import random

def retry_with_backoff(func, max_retries=3, base_delay=1.0):
    """
    THE most important SRE pattern for reliability.
    Retry with exponential backoff + jitter.
    """
    for attempt in range(1, max_retries + 1):
        try:
            return func()                      # success!
        except Exception as e:
            if attempt == max_retries:
                print(f"All {max_retries} attempts failed")
                raise
            
            # Exponential backoff: 1s, 2s, 4s, 8s...
            delay = base_delay * (2 ** (attempt - 1))
            # Jitter: add randomness to avoid thundering herd
            jitter = random.uniform(0, delay * 0.1)
            total = delay + jitter
            
            print(f"Attempt {attempt} failed: {e}. Retry in {total:.1f}s")
            time.sleep(total)

# Usage:
def check_api():
    if random.random() < 0.7:       # 70% failure rate
        raise ConnectionError("API timeout")
    return {"status": "ok"}

try:
    result = retry_with_backoff(check_api, max_retries=5)
    print(f"Success: {result}")
except ConnectionError:
    print("API completely down, escalating...")
```

> **WHY BACKOFF + JITTER?**
>
> Attempt 1: fail → wait ~1s
> Attempt 2: fail → wait ~2s
> Attempt 3: fail → wait ~4s
> Attempt 4: fail → wait ~8s
> Attempt 5: fail → GIVE UP
>
> Without backoff: you hammer a dying service, making it worse.
> Without jitter: 1000 clients all retry at the same moment.

### Circuit Breaker (Simplified)

```python
class CircuitBreaker:
    """Stop calling a failing service to let it recover."""
    
    def __init__(self, threshold=5, cooldown=30):
        self.threshold = threshold
        self.cooldown = cooldown
        self.failures = 0
        self.last_failure = 0
        self.is_open = False
    
    def call(self, func):
        if self.is_open:
            elapsed = time.time() - self.last_failure
            if elapsed < self.cooldown:
                raise ServerError(f"Circuit open. Retry in {self.cooldown - elapsed:.0f}s")
            self.is_open = False
            self.failures = 0
        
        try:
            result = func()
            self.failures = 0
            return result
        except Exception:
            self.failures += 1
            self.last_failure = time.time()
            if self.failures >= self.threshold:
                self.is_open = True
                print(f"Circuit OPENED after {self.failures} failures")
            raise
```

### DAY 3 EXERCISES

> **EXERCISE 1: Build Custom Exceptions**
>
> Create a hierarchy for a monitoring system:
> a) `MonitoringError` (base)
> b) `MetricCollectionError` — with host and metric_name attributes
> c) `ThresholdExceededError` — with metric, value, threshold
> d) `AlertDeliveryError` — with alert_type and reason
> e) Write functions that raise each one
> f) Catch `MonitoringError` to handle all of them at once

> **EXERCISE 2: Retry Decorator**
>
> Write `@retry(max_attempts=3, delay=1)`:
> a) Catches any exception and retries
> b) Waits between attempts
> c) After max_attempts, re-raises
> d) Prints attempt number on each failure
> e) Test with a randomly failing function

> **EXERCISE 3: Error Report Generator**
>
> Process a list of operations, track results:
> a) Each operation is a dict: {"type": "parse", "data": "input"}
> b) Execute each in try/except
> c) Track successes, failures, error types
> d) Print summary: "7/10 success, 3 failed (ValueError: 2, KeyError: 1)"

> **DRILL: 10 SRE Error Patterns (3 min each)**
>
> 1. Create a custom exception with extra attributes
> 2. Build a 3-level exception hierarchy
> 3. Catch the parent to handle all children
> 4. Write retry logic (3 attempts, 1s delay)
> 5. Explain exponential backoff in your own words
> 6. Write a function that raises different exceptions based on input
> 7. Use `raise ... from e` for chaining
> 8. Write a function that NEVER crashes
> 9. What's wrong with `except Exception: pass`?
> 10. When SHOULD you let an exception crash the program?

---

## DAY 4: FILE I/O & CONTEXT MANAGERS
**May 27, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: reading/writing files, with statement, pathlib
> 00:20-01:10 Hands-on: file operations for SRE
> 01:10-01:30 Drills: file challenges

### Reading Files

```python
# ALWAYS use 'with' — it closes the file automatically

# Read entire file:
with open("server.log") as f:
    content = f.read()

# Read lines into list:
with open("server.log") as f:
    lines = f.readlines()        # ['line1\n', 'line2\n', ...]

# Read line by line (memory efficient):
with open("server.log") as f:
    for line in f:
        line = line.strip()      # remove \n
        print(line)

# Safe reading (SRE way):
def read_file_safe(filepath, default=""):
    try:
        with open(filepath) as f:
            return f.read().strip()
    except FileNotFoundError:
        print(f"Not found: {filepath}")
        return default
    except PermissionError:
        print(f"Permission denied: {filepath}")
        return default
```

### Writing Files

```python
# Write (creates or OVERWRITES):
with open("output.txt", "w") as f:
    f.write("Hello, World!\n")
    f.write("Second line\n")

# Append (adds to end):
with open("output.txt", "a") as f:
    f.write("Appended line\n")

# Write with print() — handy trick:
with open("report.txt", "w") as f:
    print("=== Server Report ===", file=f)
    print(f"Host: web-01", file=f)
    print(f"CPU: 45.2%", file=f)

# Write multiple lines:
lines = ["server: web-01", "cpu: 45.2", "status: healthy"]
with open("report.txt", "w") as f:
    for line in lines:
        f.write(line + "\n")
```

### pathlib — Modern File Paths

```python
from pathlib import Path

# Create paths:
log_dir = Path("/var/log")
config = Path("/etc/myapp/config.yml")
home = Path.home()

# Join paths (/ operator):
app_log = log_dir / "myapp" / "app.log"    # /var/log/myapp/app.log

# Check existence:
config.exists()                  # True/False
config.is_file()                 # True if file
log_dir.is_dir()                 # True if directory

# Get parts:
config.name                      # 'config.yml'
config.stem                      # 'config'
config.suffix                    # '.yml'
config.parent                    # Path('/etc/myapp')

# Read/write shortcuts:
text = config.read_text()
config.write_text("new content")

# List/find files:
for item in Path("/var/log").iterdir():
    print(item.name)

log_files = list(Path("/var/log").glob("*.log"))        # all .log
all_logs = list(Path("/var/log").rglob("*.log"))        # recursive

# Create directories:
Path("output/reports/2025").mkdir(parents=True, exist_ok=True)
```

### Context Managers

```python
# 'with' ensures cleanup even on errors.

# Without with (BAD):
f = open("data.txt")
try:
    data = f.read()
finally:
    f.close()

# With 'with' (GOOD):
with open("data.txt") as f:
    data = f.read()
# f is closed automatically here

# Multiple files:
with open("input.txt") as infile, open("output.txt", "w") as outfile:
    for line in infile:
        outfile.write(line.upper())

# Custom context manager:
from contextlib import contextmanager

@contextmanager
def timer(label):
    """Time a block of code."""
    import time
    start = time.time()
    print(f"Starting: {label}")
    yield
    elapsed = time.time() - start
    print(f"Finished: {label} ({elapsed:.2f}s)")

with timer("Process logs"):
    for i in range(1000000):
        pass
```

### SRE File Patterns

```python
# Pattern 1: Log parser
def parse_log_file(filepath):
    entries, errors = [], []
    try:
        with open(filepath) as f:
            for num, line in enumerate(f, 1):
                line = line.strip()
                if not line:
                    continue
                try:
                    parts = line.split(None, 3)
                    entries.append({
                        "date": parts[0], "time": parts[1],
                        "level": parts[2], "message": parts[3] if len(parts) > 3 else ""
                    })
                except (IndexError, ValueError) as e:
                    errors.append(f"Line {num}: {e}")
    except FileNotFoundError:
        return [], [f"File not found: {filepath}"]
    return entries, errors

# Pattern 2: Atomic write (all-or-nothing)
import os
def safe_write(filepath, content):
    tmp = filepath + ".tmp"
    try:
        with open(tmp, "w") as f:
            f.write(content)
            f.flush()
        os.rename(tmp, filepath)       # atomic on most filesystems
    except Exception:
        if os.path.exists(tmp):
            os.remove(tmp)
        raise

# Pattern 3: Tail (last N lines)
def tail(filepath, n=10):
    with open(filepath) as f:
        return f.readlines()[-n:]
```

### DAY 4 EXERCISES

> **EXERCISE 1: Log File Analyzer**
>
> a) Create a sample log file (10 lines, mixed levels)
> b) Read line by line, count per level (INFO, WARN, ERROR)
> c) Extract ERROR messages into a list
> d) Write a summary report to "log_report.txt"
> e) Handle FileNotFoundError gracefully

> **EXERCISE 2: Config File Manager Class**
>
> Build `ConfigManager`:
> a) `load(filepath)` — reads key=value into dict
> b) `get(key, default=None)` — safe access
> c) `set(key, value)` — update a value
> d) `save(filepath)` — write back to file
> e) Skip comments (#) and blank lines
> f) Handle all file errors gracefully

> **EXERCISE 3: pathlib Practice**
>
> a) Create directory: `output/logs/2025/05/`
> b) Write 3 log files into it
> c) Use glob to find all .log files
> d) Print each file's name, size, parent
> e) Combine all into "combined.log"

> **EXERCISE 4: Timer Context Manager Class**
>
> Create using `__enter__` and `__exit__` (not decorator):
> a) Record start time in __enter__, return self
> b) Record end time in __exit__
> c) Store elapsed as attribute
> d) Test: `with Timer("Sort") as t: sorted(range(100000))`

> **DRILL: 10 File I/O Challenges (3 min each)**
>
> 1. Read a file, print line count
> 2. Write a list of strings to file, one per line
> 3. Append a line to existing file
> 4. Print only lines containing "ERROR"
> 5. Use pathlib to check file existence
> 6. Use glob to find all .py files
> 7. Read file → reverse lines → write to new file
> 8. Create a context manager that prints "START"/"END"
> 9. What happens opening nonexistent file for reading?
> 10. What happens writing to file in nonexistent directory?

---

## DAY 5: TESTING WITH PYTEST & CAPSTONE
**May 28, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: pytest basics, testing exceptions
> 00:20-01:10 Hands-on: capstone with tests
> 01:10-01:30 Drills: testing challenges

### Why Test?

```
Your SRE script runs at 3 AM during an incident.
Does it still work after that library update?
Does it handle the edge case that just appeared in prod?
TESTS answer these questions BEFORE production finds out.
```

### pytest Basics

```python
# Install: pip install pytest
# File names: test_*.py
# Function names: test_*

# File: test_basics.py

def add(a, b):
    return a + b

def test_add_positive():
    assert add(2, 3) == 5

def test_add_negative():
    assert add(-1, -1) == -2

def test_add_zero():
    assert add(0, 0) == 0

# Run: python3 -m pytest test_basics.py -v
```

### Testing Exceptions

```python
import pytest

def parse_port(value):
    port = int(value)
    if port < 1 or port > 65535:
        raise ValueError(f"Port out of range: {port}")
    return port

def test_parse_valid():
    assert parse_port("8080") == 8080

def test_parse_invalid():
    with pytest.raises(ValueError):
        parse_port("hello")

def test_parse_out_of_range():
    with pytest.raises(ValueError, match="out of range"):
        parse_port("99999")

# Access exception details:
def test_error_message():
    with pytest.raises(ValueError) as exc_info:
        parse_port("99999")
    assert "99999" in str(exc_info.value)
```

### Testing Classes

```python
class Server:
    def __init__(self, name, ip):
        if not name:
            raise ValueError("Name required")
        self.name = name
        self.ip = ip
        self.cpu = 0.0
    
    def check_health(self):
        if self.cpu > 90: return "critical"
        if self.cpu > 80: return "warning"
        return "healthy"

def test_creation():
    s = Server("web-01", "10.0.1.1")
    assert s.name == "web-01"
    assert s.cpu == 0.0

def test_empty_name():
    with pytest.raises(ValueError):
        Server("", "10.0.1.1")

def test_healthy():
    s = Server("web-01", "10.0.1.1")
    s.cpu = 50.0
    assert s.check_health() == "healthy"

def test_warning():
    s = Server("web-01", "10.0.1.1")
    s.cpu = 85.0
    assert s.check_health() == "warning"

def test_critical():
    s = Server("web-01", "10.0.1.1")
    s.cpu = 95.0
    assert s.check_health() == "critical"

# BOUNDARY TESTS (important for interviews!):
def test_boundary_80():
    s = Server("web-01", "10.0.1.1")
    s.cpu = 80.0
    assert s.check_health() == "healthy"    # 80 is NOT > 80

def test_boundary_90():
    s = Server("web-01", "10.0.1.1")
    s.cpu = 90.0
    assert s.check_health() == "warning"    # 90 is NOT > 90
```

### pytest Fixtures

```python
import pytest

@pytest.fixture
def web_server():
    """Reusable test setup."""
    s = Server("web-01", "10.0.1.1")
    s.cpu = 45.0
    return s

@pytest.fixture
def critical_server():
    s = Server("db-01", "10.0.2.1")
    s.cpu = 95.0
    return s

# Use fixtures as parameters:
def test_web_healthy(web_server):
    assert web_server.check_health() == "healthy"

def test_critical(critical_server):
    assert critical_server.check_health() == "critical"
```

### CAPSTONE: Log Analyzer with Tests

> **Build two files:**
>
> **log_analyzer.py:**
> ```
> class LogEntry:
>     __init__(timestamp, level, host, message)
>     is_error() → bool
>     matches_host(hostname) → bool
>     __str__()
>
> class LogParseError(Exception):
>     __init__(line_number, line_text, reason)
>
> class LogAnalyzer:
>     __init__()
>     parse_line(line) → LogEntry (raises LogParseError)
>     load_file(filepath) → (success_count, error_count)
>     count_by_level() → dict
>     get_errors() → list[LogEntry]
>     get_entries_for_host(host) → list
>     error_rate() → float
>     summary() → print report
>     save_report(filepath)
> ```
>
> **test_log_analyzer.py:**
> - Test parse_line with valid/invalid input
> - Test count_by_level returns correct counts
> - Test get_errors returns only ERRORs
> - Test error_rate (including zero division)
> - Test load_file with missing file
> - Test boundary/edge cases
> - Use at least 2 fixtures
>
> Run: `python3 -m pytest test_log_analyzer.py -v`

### DAY 5 EXERCISES

> **EXERCISE 1: Test Previous Code**
>
> Pick a class from this week:
> a) Write 8+ tests for normal behavior
> b) Write 4+ tests for error cases
> c) Write 2+ boundary tests
> d) Use at least 1 fixture
> e) All tests passing with pytest

> **EXERCISE 2: TDD (Test-Driven Development)**
>
> Write tests FIRST, then code:
> a) Tests for `validate_ip(ip_string)`:
>    - True: "10.0.1.1", "255.255.255.0"
>    - False: "256.0.0.1", "1.2.3", "hello", ""
> b) Run tests — they FAIL (no function yet)
> c) Write the function to make tests pass
> d) This is TDD — a real professional practice

> **EXERCISE 3: Integration Test**
>
> Write a test that:
> a) Creates a temp log file using Python
> b) Creates a LogAnalyzer
> c) Loads the file
> d) Checks count_by_level, get_errors, error_rate
> e) Cleans up the temp file in finally or with fixture

> **DRILL: 10 Testing Challenges (3 min each)**
>
> 1. Write a test using assert
> 2. Test that an exception is raised with pytest.raises
> 3. Use `match` parameter in pytest.raises
> 4. Create and use a pytest fixture
> 5. Test a boundary value
> 6. What does `pytest -v` show?
> 7. Test a function returns a list of correct length
> 8. Test a function returns None
> 9. Test __eq__ on two objects
> 10. Explain TDD in one sentence

---

## WEEK 15 SUMMARY

> **FILE 10 COMPLETE — ERROR HANDLING & FILE I/O**
>
> ✓ Exceptions: types, tracebacks, catching specific errors
> ✓ try/except/else/finally: when each block runs
> ✓ Raising: raise, re-raise, chaining with `from e`
> ✓ Custom exceptions: hierarchy, extra attributes, real-world design
> ✓ SRE patterns: retry + backoff, circuit breaker, graceful degradation
> ✓ File I/O: read, write, append, pathlib, glob
> ✓ Context managers: with statement, custom context managers
> ✓ pytest: assert, pytest.raises, fixtures, TDD
> ✓ 40+ exercises including capstone with tests
>
> **INTERVIEW PREP:**
> - "Difference between bare except and except Exception?"
>   (bare catches KeyboardInterrupt/SystemExit too)
> - "What is exponential backoff?" (delay doubles each retry + jitter)
> - "What is a context manager?" (cleanup via __enter__/__exit__)
> - "How to test exceptions?" (pytest.raises or try/except + assert)
> - "What are custom exceptions for?" (clarity, hierarchy, extra data)
>
> Next: File 11 — Python Advanced I (Week 16, May 31 - Jun 4)
> Topics: Decorators, Generators, Iterators, Context Managers (advanced)
