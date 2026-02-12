# PYTHON TRACK — FILE 09 of 48
## PYTHON OOP (Object-Oriented Programming)
### Classes, Objects, Methods, Inheritance, Dunder Methods, Real-World Patterns
**Week 14 | May 17 - May 21, 2025 | 5 Days | 1.5-2 hrs/day**

---

> **HONEST TALK ABOUT OOP**
>
> Most tutorials teach OOP by saying "a class is like a blueprint" and then
> show you a Dog class with bark() and sit() methods. That's useless.
> You still don't know WHY you'd use OOP in real work.
>
> So let's start differently. Let's start with PAIN.
> Let's see what life looks like WITHOUT OOP, feel the frustration,
> and then let OOP rescue us. Once you feel the "aha", it sticks.
>
> **Take your time with this file.** OOP is a mindset shift.
> If a day takes 2 hours instead of 1.5, that's perfectly fine.

| Day | Topic | Key Skills |
|-----|-------|-----------|
| 1 | WHY OOP? (The Pain, Then the Fix) | The real problem OOP solves |
| 2 | Building Real Classes | __init__, methods, self, attributes |
| 3 | Leveling Up Classes | Properties, class methods, static methods |
| 4 | Inheritance & Composition | Reuse code, build hierarchies |
| 5 | Dunder Methods & Capstone | __str__, __repr__, __eq__, full project |

---

## DAY 1: WHY OOP? — THE PAIN, THEN THE FIX
**May 17, 2025 | 1.5-2 hrs**

> 00:00-00:30 Teach: life without OOP, then life with OOP
> 00:30-01:10 Hands-on: convert messy code to classes
> 01:10-01:30 Drills: first class exercises

### Step 1: Life WITHOUT OOP (Feel the Pain)

Let's say you're an SRE monitoring 3 servers. Without OOP, you'd do something like this:

```python
# ========================================
# APPROACH 1: Separate variables (terrible)
# ========================================

server1_name = "web-01"
server1_ip = "10.0.1.1"
server1_cpu = 45.2
server1_memory = 62.0
server1_status = "healthy"

server2_name = "web-02"
server2_ip = "10.0.1.2"
server2_cpu = 87.5
server2_memory = 71.0
server2_status = "degraded"

server3_name = "db-01"
server3_ip = "10.0.2.1"
server3_cpu = 92.0
server3_memory = 88.0
server3_status = "critical"

# Want to check health? Write the same logic 3 times:
if server1_cpu > 90:
    print(f"ALERT: {server1_name} CPU critical!")
if server2_cpu > 90:
    print(f"ALERT: {server2_name} CPU critical!")
if server3_cpu > 90:
    print(f"ALERT: {server3_name} CPU critical!")

# Add a 4th server? Copy-paste EVERYTHING again.
# 50 servers? Nightmare.
```

That's painful. Let's try dictionaries:

```python
# ========================================
# APPROACH 2: Dictionaries (better, but...)
# ========================================

server1 = {"name": "web-01", "ip": "10.0.1.1", "cpu": 45.2, "memory": 62.0, "status": "healthy"}
server2 = {"name": "web-02", "ip": "10.0.1.2", "cpu": 87.5, "memory": 71.0, "status": "degraded"}
server3 = {"name": "db-01",  "ip": "10.0.2.1", "cpu": 92.0, "memory": 88.0, "status": "critical"}

servers = [server1, server2, server3]

# Check health — at least we can loop now:
def check_health(server):
    if server["cpu"] > 90:
        return "critical"
    elif server["cpu"] > 80:
        return "warning"
    return "healthy"

for s in servers:
    status = check_health(s)
    print(f"{s['name']}: {status}")
```

This is better! But there are problems:

```python
# PROBLEM 1: No protection — anyone can put garbage in
server1["cpu"] = "banana"        # Python doesn't stop you!
server1["cpuuuu"] = 50           # typo creates a new key silently

# PROBLEM 2: Functions are separate from data
# check_health() works on servers, but nothing CONNECTS them
# As your code grows, you have functions scattered everywhere

# PROBLEM 3: No consistency
# What keys should a server have? There's no template.
# Someone creates: {"hostname": "web-04", "cpu_pct": 50}
# Different key names! Your check_health() breaks.

# PROBLEM 4: Can't add behaviors
# What if you want a server to "restart itself" or "run diagnostics"?
# You'd need separate functions: restart_server(server), diagnose(server)
# Nothing ties them to the server concept.
```

### Step 2: OOP Solves ALL of This

```python
# ========================================
# APPROACH 3: A CLASS (the OOP way)
# ========================================

class Server:
    """Represents a server in our infrastructure."""

    def __init__(self, name, ip, cpu=0.0, memory=0.0):
        """This runs automatically when you create a Server."""
        self.name = name           # these are ATTRIBUTES
        self.ip = ip               # data that belongs to this server
        self.cpu = cpu
        self.memory = memory

    def check_health(self):
        """Determine server health based on metrics."""
        if self.cpu > 90:
            return "critical"
        elif self.cpu > 80:
            return "warning"
        return "healthy"

    def summary(self):
        """Print a one-line summary."""
        status = self.check_health()
        print(f"{self.name} ({self.ip}) | CPU: {self.cpu}% | Mem: {self.memory}% | [{status.upper()}]")
```

Now let's USE it:

```python
# Create server OBJECTS (instances of the Server class):
web1  = Server("web-01", "10.0.1.1", cpu=45.2, memory=62.0)
web2  = Server("web-02", "10.0.1.2", cpu=87.5, memory=71.0)
db1   = Server("db-01",  "10.0.2.1", cpu=92.0, memory=88.0)

# Access attributes:
print(web1.name)           # 'web-01'
print(web1.cpu)            # 45.2

# Call methods (functions that belong to the object):
print(web1.check_health()) # 'healthy'
print(db1.check_health())  # 'critical'

# Print summaries:
web1.summary()             # web-01 (10.0.1.1) | CPU: 45.2% | Mem: 62.0% | [HEALTHY]
db1.summary()              # db-01 (10.0.2.1) | CPU: 92.0% | Mem: 88.0% | [CRITICAL]

# Loop through multiple servers:
servers = [web1, web2, db1]
for server in servers:
    server.summary()

# Update a value:
web2.cpu = 55.0
print(web2.check_health())  # 'healthy' (was warning before)
```

### Why Is This Better? Let's Count the Wins:

```
✓ DATA + BEHAVIOR live together
  server.check_health() — the function BELONGS to the server

✓ CONSISTENT STRUCTURE
  Every Server has name, ip, cpu, memory — guaranteed by __init__

✓ REUSABLE
  Server("web-04", "10.0.1.4") — create as many as you want

✓ READABLE
  server.summary() instead of print_server_summary(server_dict)

✓ MAINTAINABLE
  Add a feature? Add ONE method. All servers get it instantly.
```

### The Key Terms (Now They'll Make Sense)

```
CLASS       = The TEMPLATE (blueprint) for creating objects
             Example: Server is the class

OBJECT      = A specific INSTANCE created from the class
             Example: web1 = Server("web-01", "10.0.1.1") — web1 is an object

ATTRIBUTE   = Data stored in an object (variables inside the object)
             Example: web1.name, web1.cpu

METHOD      = A function that belongs to an object
             Example: web1.check_health(), web1.summary()

self        = Refers to "this specific object" inside the class
             When you call web1.check_health(), self IS web1

__init__    = The CONSTRUCTOR — runs automatically when you create an object
             Sets up the object's initial attributes
```

### What is `self`? (The #1 Confusion)

```python
# self is NOT magic. It's just a reference to the current object.

class Server:
    def __init__(self, name, ip):
        # self.name = name means:
        # "Store 'name' inside THIS specific object"
        self.name = name
        self.ip = ip

    def greet(self):
        # self.name means:
        # "Get the name of THIS specific object"
        print(f"I am {self.name} at {self.ip}")

# When you call:
web1 = Server("web-01", "10.0.1.1")
web1.greet()

# Python secretly does this:
# Server.greet(web1)      ← web1 becomes 'self'

# So self IS web1 when you call web1.greet()
# And self IS web2 when you call web2.greet()

web2 = Server("web-02", "10.0.1.2")
web2.greet()               # I am web-02 at 10.0.1.2  ← self is web2 here
```

> **STILL CONFUSED ABOUT self?**
>
> Think of it as "me" in English.
>
> If a server could talk:
> - "My name is web-01" → `self.name`
> - "My CPU is 45%" → `self.cpu`
> - "Let me check my health" → `self.check_health()`
>
> Every method needs `self` as the first parameter so it knows
> WHICH object it's working on. Python passes it automatically.

### DAY 1 HANDS-ON: Convert Dict Code to Classes

> **EXERCISE 1: Your First Class — Step by Step**
>
> Follow along and TYPE every line (don't copy-paste):
>
> a) Create a file called `server.py`
> b) Define a class `Server` with `__init__` that takes: name, ip, port
> c) Add a method `info()` that prints: "Server web-01 at 10.0.1.1:8080"
> d) Create 3 server objects
> e) Call `.info()` on each
> f) Store all 3 in a list, loop through and call `.info()`
>
> **Starter code:**
> ```python
> class Server:
>     def __init__(self, name, ip, port):
>         # YOUR CODE: store name, ip, port as attributes
>         pass
>
>     def info(self):
>         # YOUR CODE: print server info using self.name, etc.
>         pass
>
> # Create servers and test:
> web1 = Server("web-01", "10.0.1.1", 8080)
> web1.info()    # Should print: Server web-01 at 10.0.1.1:8080
> ```

> **EXERCISE 2: Add Methods**
>
> Extend your Server class:
> a) Add `cpu` and `memory` attributes (with defaults of 0.0 in __init__)
> b) Add `check_health()` method that returns "critical"/"warning"/"healthy"
> c) Add `update_metrics(cpu, memory)` method that updates the values
> d) Add `summary()` method that prints a formatted line with all info
> e) Create 4 servers, update their metrics, print summaries

> **EXERCISE 3: Compare With and Without**
>
> First write this WITHOUT OOP (using dicts):
> - Track 3 employees: name, department, salary
> - Write a function to give a 10% raise
> - Write a function to print employee info
>
> Then rewrite it WITH a class:
> - Employee class with name, department, salary
> - give_raise(percent) method
> - info() method
>
> Compare: which is cleaner? Which would scale better to 100 employees?

> **EXERCISE 4: Build a Counter Class**
>
> Create a `Counter` class that:
> a) Starts at 0
> b) Has an `increment()` method that adds 1
> c) Has a `decrement()` method that subtracts 1
> d) Has a `reset()` method that goes back to 0
> e) Has a `value()` method that returns the current count
> f) Test it: increment 5 times, decrement 2 times, print value (should be 3)

> **DRILL: 10 Quick Class Exercises (3 min each)**
>
> 1. Create a `Dog` class with name and breed. Add a `bark()` method that prints "Woof!"
> 2. Create a `Rectangle` class with width and height. Add an `area()` method.
> 3. Create a `Temperature` class that stores celsius. Add a `to_fahrenheit()` method.
> 4. Create a `BankAccount` class with balance. Add `deposit(amount)` and `withdraw(amount)`.
> 5. What happens if you forget `self` in a method definition? Try it.
> 6. What happens if you forget `self.` when accessing an attribute? Try it.
> 7. Create 2 objects from the same class. Change one's attribute. Does the other change?
> 8. Can you add an attribute AFTER creating an object? (e.g., `web1.new_thing = "hello"`)
> 9. Create a `Person` class with a `greet(other_name)` method that prints "Hi {other_name}, I'm {self.name}"
> 10. Create a list of 5 Rectangle objects with different sizes. Find the one with largest area.

---

## DAY 2: BUILDING REAL CLASSES
**May 18, 2025 | 1.5-2 hrs**

> 00:00-00:30 Teach: deeper __init__, default values, validation, multiple methods
> 00:30-01:10 Hands-on: build a real SRE tool class
> 01:10-01:30 Drills: class design challenges

### Smarter __init__ with Defaults and Validation

```python
class Server:
    def __init__(self, name, ip, port=8080, environment="production"):
        """
        Create a new Server.

        Args:
            name: hostname (required)
            ip: IP address (required)
            port: listening port (default 8080)
            environment: prod/staging/dev (default production)
        """
        # Validation — reject bad data immediately
        if not name:
            raise ValueError("Server name cannot be empty")
        if port < 1 or port > 65535:
            raise ValueError(f"Invalid port: {port}")
        if environment not in ("production", "staging", "dev"):
            raise ValueError(f"Invalid environment: {environment}")

        # Store attributes
        self.name = name
        self.ip = ip
        self.port = port
        self.environment = environment
        self.cpu = 0.0
        self.memory = 0.0
        self.is_healthy = True
        self.request_count = 0
        self.error_count = 0

# Use defaults:
web1 = Server("web-01", "10.0.1.1")              # port=8080, env=production
web2 = Server("web-02", "10.0.1.2", port=9090)   # override port
db1  = Server("db-01", "10.0.2.1", port=5432, environment="staging")

# Validation catches bad data:
# bad = Server("", "10.0.1.1")        # ValueError: name cannot be empty
# bad = Server("x", "y", port=99999)  # ValueError: Invalid port
```

### Methods That Work Together

```python
class Server:
    def __init__(self, name, ip, port=8080):
        self.name = name
        self.ip = ip
        self.port = port
        self.cpu = 0.0
        self.memory = 0.0
        self.request_count = 0
        self.error_count = 0
        self._alerts = []           # underscore = "private" (convention)

    def update_metrics(self, cpu, memory):
        """Update CPU and memory, auto-check for alerts."""
        self.cpu = cpu
        self.memory = memory
        self._evaluate_health()     # methods can call other methods!

    def record_request(self, success=True):
        """Record an incoming request."""
        self.request_count += 1
        if not success:
            self.error_count += 1

    def error_rate(self):
        """Calculate error rate as a percentage."""
        if self.request_count == 0:
            return 0.0
        return (self.error_count / self.request_count) * 100

    def _evaluate_health(self):
        """Internal method to check metrics and create alerts."""
        # Leading underscore = "don't call this from outside"
        # It's a CONVENTION, not enforced
        self._alerts = []
        if self.cpu > 90:
            self._alerts.append(f"CPU critical: {self.cpu}%")
        elif self.cpu > 80:
            self._alerts.append(f"CPU warning: {self.cpu}%")
        if self.memory > 90:
            self._alerts.append(f"Memory critical: {self.memory}%")

    def get_alerts(self):
        """Return current alerts."""
        return self._alerts

    def summary(self):
        """Full status summary."""
        status = "CRITICAL" if self.cpu > 90 or self.memory > 90 else \
                 "WARNING" if self.cpu > 80 or self.memory > 80 else "OK"
        err = self.error_rate()
        print(f"{self.name:<10} CPU:{self.cpu:>5.1f}%  Mem:{self.memory:>5.1f}%  "
              f"Err:{err:>5.1f}%  [{status}]")
        for alert in self._alerts:
            print(f"  └─ {alert}")

# See it in action:
web1 = Server("web-01", "10.0.1.1")
web1.update_metrics(cpu=87.5, memory=62.0)    # triggers _evaluate_health
web1.record_request(success=True)
web1.record_request(success=True)
web1.record_request(success=False)
web1.summary()
# web-01     CPU: 87.5%  Mem: 62.0%  Err: 33.3%  [WARNING]
#   └─ CPU warning: 87.5%
```

> **KEY INSIGHT: METHODS CALLING METHODS**
>
> Notice how `update_metrics()` calls `_evaluate_health()`.
> This is normal! Methods inside a class can call each other using `self.`
>
> This is the power of OOP: you build small pieces, then combine them.
> Each method does ONE thing. Together they do something complex.

### The Underscore Convention

```python
# Python doesn't have true "private" like Java/C++
# Instead, we use naming conventions:

class Example:
    def __init__(self):
        self.public = "anyone can access"       # no underscore
        self._internal = "please don't touch"   # single underscore = internal
        self.__mangled = "harder to access"      # double underscore = name mangling

# Single underscore (_) = "This is internal. You CAN access it, but you shouldn't."
# It's like a sign that says "Staff Only" — not a locked door.

# In practice, single underscore is what you'll use 99% of the time.
# Double underscore is rare. Don't worry about it for now.
```

### DAY 2 HANDS-ON: Build a Log Entry Class

> **EXERCISE 1: LogEntry Class (Build Step by Step)**
>
> Create a class that represents a single log line:
> ```python
> class LogEntry:
>     def __init__(self, timestamp, level, host, message):
>         # Store all attributes
>         # Validate: level must be one of: INFO, WARN, ERROR, DEBUG
>         pass
>
>     def is_error(self):
>         # Return True if level is ERROR
>         pass
>
>     def matches_host(self, hostname):
>         # Return True if this entry is from the given host
>         pass
>
>     def format(self):
>         # Return formatted string: "[2025-05-18 ERROR] web-01: Connection refused"
>         pass
> ```
> Test with:
> ```python
> entry = LogEntry("2025-05-18 14:30:22", "ERROR", "web-01", "Connection refused")
> print(entry.is_error())           # True
> print(entry.matches_host("web-01"))  # True
> print(entry.format())             # [2025-05-18 14:30:22 ERROR] web-01: Connection refused
> ```

> **EXERCISE 2: LogAnalyzer Class**
>
> Build a class that analyzes a COLLECTION of LogEntry objects:
> ```python
> class LogAnalyzer:
>     def __init__(self):
>         self.entries = []       # list of LogEntry objects
>
>     def add_entry(self, entry):
>         pass                    # append to self.entries
>
>     def count_by_level(self):
>         pass                    # return dict: {"ERROR": 3, "INFO": 5, ...}
>
>     def get_errors(self):
>         pass                    # return list of only ERROR entries
>
>     def get_entries_for_host(self, hostname):
>         pass                    # return entries matching hostname
>
>     def summary(self):
>         pass                    # print total entries, errors, unique hosts
> ```
>
> Create 10+ LogEntry objects, add them to the analyzer, run all methods.

> **EXERCISE 3: BankAccount Class**
>
> a) Create a class with owner, balance (default 0)
> b) `deposit(amount)` — add money (reject negative amounts)
> c) `withdraw(amount)` — remove money (reject if insufficient funds)
> d) `transfer(other_account, amount)` — move money to another BankAccount
> e) `statement()` — print owner and current balance
> f) Test: create 2 accounts, deposit, withdraw, transfer between them

> **DRILL: 10 Class Building Challenges (3 min each)**
>
> 1. Create a `Stopwatch` class: start(), stop(), elapsed() methods
> 2. Create a `ShoppingCart`: add_item(name, price), total(), item_count()
> 3. Create a `Dice`: roll() returns random 1-6 (use `import random`)
> 4. Add validation: reject negative price in a Product class
> 5. Create a class where one method calls another method
> 6. Create a `Stack`: push(item), pop(), peek(), is_empty()
> 7. Create a `Playlist`: add_song(name), next_song(), current_song()
> 8. Why does `self._alerts = []` use underscore? What does it signal?
> 9. What happens if __init__ doesn't return anything? (It shouldn't!)
> 10. Create a `Config` class that stores settings as a dict and has a get(key, default) method

---

## DAY 3: LEVELING UP CLASSES
**May 19, 2025 | 1.5-2 hrs**

> 00:00-00:30 Teach: properties, class attributes, class methods, static methods
> 00:30-01:10 Hands-on: enhanced class exercises
> 01:10-01:30 Drills: advanced patterns

### Class Attributes vs Instance Attributes

```python
class Server:
    # CLASS ATTRIBUTE — shared by ALL Server objects
    total_count = 0
    all_servers = []

    def __init__(self, name, ip):
        # INSTANCE ATTRIBUTES — unique to THIS specific object
        self.name = name
        self.ip = ip
        self.cpu = 0.0

        # Update class-level tracking:
        Server.total_count += 1
        Server.all_servers.append(self)

# Create servers:
web1 = Server("web-01", "10.0.1.1")
web2 = Server("web-02", "10.0.1.2")
db1  = Server("db-01",  "10.0.2.1")

# Instance attributes are different per object:
print(web1.name)               # 'web-01'
print(web2.name)               # 'web-02'

# Class attributes are shared:
print(Server.total_count)      # 3
print(len(Server.all_servers)) # 3

# You can also access class attributes through objects:
print(web1.total_count)        # 3 (same value)
```

> **WHEN TO USE WHICH**
>
> - **Instance attribute** (self.name): data that differs per object
>   → each server has its own name, IP, CPU
>
> - **Class attribute** (Server.total_count): data shared by all objects
>   → total count of servers, default settings, constants

### Properties — Smart Attributes

```python
# Sometimes you want an attribute that DOES something when accessed.
# Properties let you add logic to getting/setting a value.

class Server:
    def __init__(self, name, ip):
        self.name = name
        self.ip = ip
        self._cpu = 0.0          # underscore: "managed by property"

    @property
    def cpu(self):
        """Get the CPU value."""
        return self._cpu

    @cpu.setter
    def cpu(self, value):
        """Set CPU with validation."""
        if not 0 <= value <= 100:
            raise ValueError(f"CPU must be 0-100, got {value}")
        self._cpu = value

    @property
    def status(self):
        """Calculated property — no setter, it's computed."""
        if self._cpu > 90:
            return "critical"
        elif self._cpu > 80:
            return "warning"
        return "healthy"

# Usage — looks like a regular attribute, but has logic behind it:
web1 = Server("web-01", "10.0.1.1")
web1.cpu = 85.0                   # calls the setter (validates!)
print(web1.cpu)                   # calls the getter → 85.0
print(web1.status)                # computed! → 'warning'

# web1.cpu = 150                  # ValueError: CPU must be 0-100
# web1.status = "healthy"         # AttributeError: can't set (no setter)
```

> **WHY PROPERTIES MATTER FOR SRE**
>
> Properties let you:
> 1. Validate data (reject bad CPU values)
> 2. Compute values on the fly (status based on current CPU)
> 3. Add logging when attributes change
> 4. Keep a simple interface: `server.cpu = 85` looks clean
>
> Without properties you'd need: `server.set_cpu(85)` and `server.get_cpu()`
> which is ugly Java-style code. Python properties are cleaner.

### Class Methods and Static Methods

```python
class Server:
    _registry = {}

    def __init__(self, name, ip, port=8080):
        self.name = name
        self.ip = ip
        self.port = port
        Server._registry[name] = self

    @classmethod
    def from_string(cls, server_string):
        """
        Create a Server from a string like "web-01:10.0.1.1:8080"
        cls = the class itself (like self, but for the class)
        """
        parts = server_string.split(":")
        name = parts[0]
        ip = parts[1]
        port = int(parts[2]) if len(parts) > 2 else 8080
        return cls(name, ip, port)       # same as Server(name, ip, port)

    @classmethod
    def get_by_name(cls, name):
        """Look up a server by name from the registry."""
        return cls._registry.get(name)

    @staticmethod
    def is_valid_ip(ip_string):
        """
        Check if a string looks like a valid IP.
        Static = doesn't need self or cls. It's just a utility function
        that's grouped with the class because it's related.
        """
        parts = ip_string.split(".")
        if len(parts) != 4:
            return False
        return all(p.isdigit() and 0 <= int(p) <= 255 for p in parts)

# Regular creation:
web1 = Server("web-01", "10.0.1.1")

# Class method — alternative constructor:
web2 = Server.from_string("web-02:10.0.1.2:9090")
print(web2.name)                   # 'web-02'
print(web2.port)                   # 9090

# Class method — registry lookup:
found = Server.get_by_name("web-01")
print(found.ip)                    # '10.0.1.1'

# Static method — utility function:
print(Server.is_valid_ip("10.0.1.1"))    # True
print(Server.is_valid_ip("999.0.1.1"))   # False
print(Server.is_valid_ip("hello"))       # False
```

> **SUMMARY: THREE TYPES OF METHODS**
>
> ```
> def method(self):      → Regular method. Has access to the object (self).
>                          Use for: anything that works with object data
>
> @classmethod
> def method(cls):       → Class method. Has access to the class (cls).
>                          Use for: alternative constructors, class-level data
>
> @staticmethod
> def method():          → Static method. No access to object or class.
>                          Use for: utility functions related to the class
> ```

### DAY 3 EXERCISES

> **EXERCISE 1: Enhanced Server Class**
>
> Build a Server class with:
> a) `_cpu` and `_memory` as internal attributes with property getters/setters
> b) Validation: both must be 0-100
> c) A computed `status` property (no setter)
> d) A class attribute `total_servers` that counts instances
> e) A `@classmethod from_dict(cls, data)` that creates from a dictionary
> f) A `@staticmethod is_valid_port(port)` that checks if port is 1-65535
>
> Test everything thoroughly.

> **EXERCISE 2: Temperature Converter**
>
> Create a Temperature class:
> a) Store temperature internally in Celsius
> b) Properties: `celsius` (getter+setter), `fahrenheit` (getter+setter), `kelvin` (getter)
> c) Setting fahrenheit should update the internal celsius value
> d) `@classmethod from_fahrenheit(cls, f)` — create from Fahrenheit
> e) `@staticmethod is_below_freezing(celsius)` — returns True if < 0
>
> ```python
> t = Temperature(100)             # 100°C
> print(t.fahrenheit)              # 212.0
> t.fahrenheit = 32                # set via fahrenheit
> print(t.celsius)                 # 0.0
> t2 = Temperature.from_fahrenheit(98.6)
> print(t2.celsius)                # 37.0
> ```

> **EXERCISE 3: Track Class Instances**
>
> Create a class `Connection` that:
> a) Has a class attribute `_all_connections = []`
> b) Each __init__ adds self to this list
> c) Has a `@classmethod get_all()` returning all connections
> d) Has a `@classmethod count()` returning total count
> e) Has a `@classmethod close_all()` that calls .close() on each
> f) Has an instance method `close()` that sets self.is_open = False

> **DRILL: 10 Advanced Class Challenges (3 min each)**
>
> 1. Create a property that returns the length of a list attribute
> 2. Create a property with a setter that only accepts positive numbers
> 3. Create a class method that returns a new object from a CSV string
> 4. Create a static method that validates an email format (contains @ and .)
> 5. What is the difference between `self.count` and `MyClass.count`?
> 6. Create a class with a `_log` list that records every method call
> 7. Make a class attribute `DEFAULT_PORT = 8080` and use it in __init__
> 8. Create a `from_json_string` classmethod (use `import json`)
> 9. Add a property that returns a formatted string (like a computed summary)
> 10. What happens if a static method tries to access self? Try it.

---

## DAY 4: INHERITANCE & COMPOSITION
**May 20, 2025 | 1.5-2 hrs**

> 00:00-00:30 Teach: inheritance, super(), overriding, composition
> 00:30-01:10 Hands-on: build a class hierarchy
> 01:10-01:30 Drills: inheritance challenges

### Inheritance — "Is-A" Relationship

```python
# Inheritance lets you create a NEW class based on an EXISTING class.
# The new class gets all the attributes and methods of the parent,
# and can add its own or override them.

# Parent class (base class):
class Server:
    def __init__(self, name, ip, port):
        self.name = name
        self.ip = ip
        self.port = port
        self.cpu = 0.0

    def check_health(self):
        if self.cpu > 90:
            return "critical"
        elif self.cpu > 80:
            return "warning"
        return "healthy"

    def summary(self):
        status = self.check_health()
        return f"{self.name} ({self.ip}:{self.port}) [{status}]"


# Child class (subclass) — WebServer IS A Server:
class WebServer(Server):
    def __init__(self, name, ip, port=80):
        super().__init__(name, ip, port)     # call parent's __init__
        self.active_connections = 0          # additional attribute
        self.max_connections = 1000

    def connection_usage(self):
        """New method — only WebServers have this."""
        return (self.active_connections / self.max_connections) * 100

    def check_health(self):
        """Override parent's check_health to add connection check."""
        base_health = super().check_health()    # call parent's version
        if self.connection_usage() > 90:
            return "critical"
        return base_health


# Child class — DatabaseServer IS A Server:
class DatabaseServer(Server):
    def __init__(self, name, ip, port=5432):
        super().__init__(name, ip, port)
        self.replication_lag_ms = 0
        self.connections = 0
        self.max_connections = 100

    def check_health(self):
        """Override with database-specific checks."""
        base_health = super().check_health()
        if self.replication_lag_ms > 5000:
            return "critical"
        if self.replication_lag_ms > 1000:
            return "warning"
        return base_health

    def summary(self):
        """Override to include replication info."""
        base = super().summary()
        return f"{base} | Repl lag: {self.replication_lag_ms}ms"
```

Using the hierarchy:

```python
# Create objects:
web = WebServer("web-01", "10.0.1.1")
db = DatabaseServer("db-01", "10.0.2.1")

# They all have Server's methods:
web.cpu = 75.0
db.cpu = 45.0
db.replication_lag_ms = 3000

print(web.summary())     # web-01 (10.0.1.1:80) [healthy]
print(db.summary())      # db-01 (10.0.2.1:5432) [warning] | Repl lag: 3000ms

# They're all "Servers":
servers = [web, db]
for s in servers:
    print(s.check_health())    # each uses its OWN version!

# Check types:
print(isinstance(web, WebServer))   # True
print(isinstance(web, Server))      # True — a WebServer IS a Server
print(isinstance(db, WebServer))    # False — a DB is NOT a WebServer
```

### super() — Calling the Parent

```python
# super() gives you access to the PARENT class.
# Use it to extend (not replace) the parent's behavior.

class DatabaseServer(Server):
    def __init__(self, name, ip, port=5432):
        super().__init__(name, ip, port)  # Let Server set up name, ip, port
        self.replication_lag_ms = 0       # Then add DB-specific stuff

# Without super().__init__(), name, ip, port would NOT be set!
# Try removing it and see what happens: AttributeError.

# super() in methods:
    def check_health(self):
        base = super().check_health()     # Get parent's answer first
        # Then add our own checks on top
        if self.replication_lag_ms > 5000:
            return "critical"
        return base
```

### Composition — "Has-A" Relationship

```python
# Not everything should use inheritance.
# Sometimes one object CONTAINS another — that's composition.

# A server HAS alerts. Alerts aren't a type of server.

class Alert:
    def __init__(self, severity, message, host):
        self.severity = severity       # "critical", "warning", "info"
        self.message = message
        self.host = host
        self.acknowledged = False

    def acknowledge(self):
        self.acknowledged = True

    def format(self):
        ack = "✓" if self.acknowledged else "✗"
        return f"[{self.severity.upper()}] {self.host}: {self.message} ({ack})"


class Server:
    def __init__(self, name, ip):
        self.name = name
        self.ip = ip
        self.cpu = 0.0
        self.alerts = []              # Server HAS a list of Alert objects

    def update_cpu(self, value):
        self.cpu = value
        if value > 90:
            # Create an Alert OBJECT and store it
            alert = Alert("critical", f"CPU at {value}%", self.name)
            self.alerts.append(alert)
        elif value > 80:
            alert = Alert("warning", f"CPU at {value}%", self.name)
            self.alerts.append(alert)

    def get_unacknowledged_alerts(self):
        return [a for a in self.alerts if not a.acknowledged]

    def show_alerts(self):
        for alert in self.alerts:
            print(f"  {alert.format()}")

# Usage:
web1 = Server("web-01", "10.0.1.1")
web1.update_cpu(95.5)
web1.update_cpu(85.0)
web1.show_alerts()
# [CRITICAL] web-01: CPU at 95.5% (✗)
# [WARNING] web-01: CPU at 85.0% (✗)

# Acknowledge an alert:
web1.alerts[0].acknowledge()
unacked = web1.get_unacknowledged_alerts()
print(f"Unacknowledged: {len(unacked)}")    # 1
```

> **INHERITANCE vs COMPOSITION: THE RULE**
>
> - **Inheritance** (IS-A): WebServer IS A Server → `class WebServer(Server)`
> - **Composition** (HAS-A): Server HAS Alerts → `self.alerts = []`
>
> **Rule of thumb:** Prefer composition over inheritance.
> Use inheritance only when there's a clear "is-a" relationship.
>
> Why? Composition is more flexible. You can swap parts in and out.
> Inheritance creates tight coupling that's hard to undo.

### DAY 4 EXERCISES

> **EXERCISE 1: Build a Server Hierarchy**
>
> Create:
> a) Base class `Server` with name, ip, cpu, memory, check_health(), summary()
> b) `WebServer(Server)` — add: active_connections, max_connections, request_rate
> c) `DatabaseServer(Server)` — add: replication_lag, query_count, is_primary
> d) `CacheServer(Server)` — add: hit_rate, eviction_count, memory_used_mb
> e) Each child overrides check_health() with type-specific checks
> f) Create one of each, put them in a list, loop and call summary()

> **EXERCISE 2: Alert System (Composition)**
>
> a) Create an `Alert` class: severity, message, timestamp, acknowledged
> b) Create an `AlertManager` class that stores a list of Alert objects:
>    - add_alert(alert)
>    - get_critical_alerts()
>    - get_unacknowledged()
>    - acknowledge_all()
>    - summary() — print counts by severity
> c) Create a `Server` class that HAS an AlertManager
> d) When server metrics cross thresholds, auto-create alerts

> **EXERCISE 3: Shape Hierarchy**
>
> (Classic exercise — good for understanding inheritance)
> a) Base class `Shape` with color attribute and `area()` method (returns 0)
> b) `Circle(Shape)` — override area() with π * r²
> c) `Rectangle(Shape)` — override area() with width * height
> d) `Square(Rectangle)` — only needs one side parameter
> e) Create a list of mixed shapes, sort by area (use sorted + key)
> f) What's the total area of all shapes?

> **DRILL: 10 Inheritance & Composition Challenges (3 min each)**
>
> 1. Create a parent Animal class with speak(). Create Dog and Cat that override it.
> 2. Use super().__init__() in a child class. Remove it and see what breaks.
> 3. Create an isinstance() check to see if an object is a specific type.
> 4. Can a child class add NEW methods that the parent doesn't have? Try it.
> 5. Create a class that contains a list of other objects (composition).
> 6. Override a method but also call the parent's version with super().
> 7. What does `type(object)` show for an instance of a child class?
> 8. Create a Vehicle → Car → ElectricCar hierarchy (3 levels).
> 9. Can a class inherit from a child? (Yes! How deep can you go?)
> 10. Create two unrelated classes. Have one CONTAIN an instance of the other.

---

## DAY 5: DUNDER METHODS & CAPSTONE
**May 21, 2025 | 2 hrs**

> 00:00-00:30 Teach: __str__, __repr__, __eq__, __lt__, __len__, __contains__
> 00:30-01:20 Hands-on: capstone project
> 01:20-02:00 Drills: full OOP review

### Dunder Methods (Double Underscore = "Magic Methods")

```python
# Dunder methods let your objects work with Python's built-in operations.
# They make your classes feel like native Python types.

# Without dunders:
web1 = Server("web-01", "10.0.1.1")
print(web1)           # <__main__.Server object at 0x7f...>  ← USELESS!
web1 == web2          # compares memory addresses, not content ← WRONG!

# With dunders — much better:

class Server:
    def __init__(self, name, ip, port=8080):
        self.name = name
        self.ip = ip
        self.port = port
        self.cpu = 0.0

    def __str__(self):
        """Human-readable string. Used by print() and str()."""
        return f"Server({self.name} @ {self.ip}:{self.port})"

    def __repr__(self):
        """Developer string. Used in debugger and REPL.
        Should ideally be valid Python to recreate the object."""
        return f'Server("{self.name}", "{self.ip}", {self.port})'

    def __eq__(self, other):
        """Equality check. Used by ==."""
        if not isinstance(other, Server):
            return False
        return self.name == other.name and self.ip == other.ip

    def __lt__(self, other):
        """Less than. Used by < and sorted()."""
        return self.cpu < other.cpu

    def __hash__(self):
        """Make objects usable in sets and as dict keys."""
        return hash((self.name, self.ip))

# Now these all work beautifully:
web1 = Server("web-01", "10.0.1.1")
web2 = Server("web-02", "10.0.1.2")
web1_copy = Server("web-01", "10.0.1.1")

print(web1)                        # Server(web-01 @ 10.0.1.1:8080)
print(repr(web2))                  # Server("web-02", "10.0.1.2", 8080)
print(web1 == web1_copy)           # True (same name and ip)
print(web1 == web2)                # False

# Sorting works because of __lt__:
web1.cpu = 85.0
web2.cpu = 45.0
servers = [web1, web2]
for s in sorted(servers):          # sorted uses __lt__
    print(f"{s.name}: {s.cpu}%")
# web-02: 45.0%
# web-01: 85.0%

# Sets work because of __hash__ and __eq__:
unique = {web1, web1_copy, web2}
print(len(unique))                 # 2 (web1 and web1_copy are "equal")
```

### More Dunder Methods

```python
class ServerGroup:
    """A group of servers — uses dunders to behave like a collection."""

    def __init__(self, name):
        self.name = name
        self._servers = []

    def add(self, server):
        self._servers.append(server)

    def __len__(self):
        """len(group) works."""
        return len(self._servers)

    def __getitem__(self, index):
        """group[0] and group[1:3] work."""
        return self._servers[index]

    def __contains__(self, server):
        """'server in group' works."""
        return server in self._servers

    def __iter__(self):
        """'for server in group' works."""
        return iter(self._servers)

    def __str__(self):
        return f"ServerGroup({self.name}: {len(self)} servers)"

# Now ServerGroup behaves like a built-in collection:
group = ServerGroup("production")
group.add(Server("web-01", "10.0.1.1"))
group.add(Server("web-02", "10.0.1.2"))
group.add(Server("db-01", "10.0.2.1"))

print(len(group))                  # 3
print(group[0])                    # Server(web-01 @ 10.0.1.1:8080)
print(group[0:2])                  # first two servers

web1 = Server("web-01", "10.0.1.1")
print(web1 in group)               # True (uses __eq__)

for server in group:               # uses __iter__
    print(f"  {server}")
```

> **THE MOST IMPORTANT DUNDERS FOR SRE**
>
> ```
> __str__      → print(obj)         Human-readable output
> __repr__     → repr(obj)          Debug output, recreatable string
> __eq__       → obj1 == obj2       Compare objects by content
> __lt__       → sorted(list)       Sort objects
> __len__      → len(obj)           Size/count
> __contains__ → item in obj        Membership check
> __iter__     → for x in obj       Looping
> __getitem__  → obj[index]         Indexing
> ```
>
> You won't need all of these all the time. Start with __str__ and __repr__
> — add those to EVERY class you write. Add others as needed.

### CAPSTONE PROJECT: Infrastructure Monitor

> **BUILD THIS FROM SCRATCH — Use everything from this week!**
>
> Create a mini infrastructure monitoring tool with these classes:
>
> **Class 1: Metric**
> ```
> - Attributes: name, value, unit, timestamp
> - Method: is_critical(threshold) — returns True if value > threshold
> - Dunder: __str__ → "CPU: 87.5%"
> - Dunder: __lt__ → compare by value (for sorting)
> ```
>
> **Class 2: Server (base class)**
> ```
> - Attributes: name, ip, environment, metrics (dict of Metric objects)
> - Method: add_metric(metric)
> - Method: check_health() → "healthy"/"warning"/"critical"
> - Property: status (computed from check_health)
> - Dunder: __str__, __repr__, __eq__ (equal if same name+ip)
> ```
>
> **Class 3: WebServer(Server)**
> ```
> - Extra: active_connections, max_connections
> - Override: check_health() adds connection check
> - Extra method: connection_usage() → percentage
> ```
>
> **Class 4: DatabaseServer(Server)**
> ```
> - Extra: replication_lag_ms, is_primary
> - Override: check_health() adds replication check
> ```
>
> **Class 5: AlertManager (composition, not inheritance)**
> ```
> - Stores list of alert dicts: {severity, message, server_name, timestamp}
> - Methods: add_alert(), get_critical(), count_by_severity()
> - Dunder: __len__ → number of alerts
> ```
>
> **Class 6: Infrastructure (ties everything together)**
> ```
> - Stores list of Server objects (and subclasses)
> - Has an AlertManager
> - Methods:
>   - add_server(server)
>   - get_servers_by_env(env)
>   - get_critical_servers()
>   - dashboard() → formatted table of all servers
>   - run_health_check() → check all servers, auto-create alerts
> - Dunder: __len__, __iter__, __contains__
> ```
>
> **Test script:**
> ```python
> # Create infrastructure
> infra = Infrastructure()
>
> # Add servers
> web1 = WebServer("web-01", "10.0.1.1", environment="prod")
> web1.add_metric(Metric("cpu", 87.5, "%"))
> web1.add_metric(Metric("memory", 62.0, "%"))
> web1.active_connections = 850
>
> db1 = DatabaseServer("db-01", "10.0.2.1", environment="prod")
> db1.add_metric(Metric("cpu", 45.0, "%"))
> db1.replication_lag_ms = 3500
>
> infra.add_server(web1)
> infra.add_server(db1)
>
> # Run checks
> infra.run_health_check()
> infra.dashboard()
>
> # Show alerts
> print(f"\nAlerts: {len(infra.alert_manager)}")
> for alert in infra.alert_manager.get_critical():
>     print(f"  {alert}")
> ```

### DAY 5 EXERCISES

> **EXERCISE 1: Make Your Classes Printable**
>
> Go back to ANY class you built this week.
> a) Add `__str__` that gives a nice human-readable string
> b) Add `__repr__` that gives a developer-friendly string
> c) Add `__eq__` that compares by the right attributes
> d) Test: print the object, compare two objects, put objects in a set

> **EXERCISE 2: Sortable Collection**
>
> Create a `Task` class with name, priority (1-5), due_date:
> a) Add `__lt__` to sort by priority
> b) Create 10 tasks with random priorities
> c) Sort them using `sorted(tasks)`
> d) Also implement `__len__` on a TaskList class
> e) Implement `__getitem__` so you can do `task_list[0]`

> **EXERCISE 3: Full OOP Review**
>
> Without looking at notes, build a `Library` system:
> a) `Book` class: title, author, isbn, is_available
> b) `Member` class: name, member_id, borrowed_books (list of Book objects)
> c) `Library` class: name, books (list), members (list)
>    - add_book(), add_member()
>    - checkout(member_id, isbn)
>    - return_book(member_id, isbn)
>    - search(title_keyword)
>    - available_books()
> d) Add __str__ to all classes
> e) Test with 5 books and 2 members

> **DRILL: 10 Final OOP Challenges (3 min each)**
>
> 1. Add __str__ to any class — print the object
> 2. Add __eq__ — test with ==
> 3. Add __len__ to a class that has a list attribute
> 4. Make a class iterable with __iter__
> 5. What's the difference between __str__ and __repr__?
> 6. Create a class, make two objects, put them in a set (needs __hash__)
> 7. Sort a list of objects by an attribute (implement __lt__)
> 8. Use isinstance() to check object types in a mixed list
> 9. Create a class that uses both inheritance AND composition
> 10. What dunder method does `in` use? (`__contains__`)

---

## WEEK 14 SUMMARY & OOP CHEAT SHEET

> **FILE 09 COMPLETE — PYTHON OOP**
>
> ✓ Day 1: WHY OOP — saw the pain without it, then the fix
> ✓ Day 2: Real classes — __init__, methods, validation, private convention
> ✓ Day 3: Properties, class/instance attributes, @classmethod, @staticmethod
> ✓ Day 4: Inheritance (is-a), composition (has-a), super()
> ✓ Day 5: Dunder methods, capstone infrastructure monitor
> ✓ 40+ exercises with SRE context throughout

### OOP QUICK REFERENCE

```
CLASS STRUCTURE:
┌─────────────────────────────────────────┐
│ class MyClass:                          │
│     class_attr = "shared"     ← shared  │
│                                         │
│     def __init__(self, ...):  ← setup   │
│         self.attr = value     ← data    │
│                                         │
│     def method(self):         ← behavior│
│         return self.attr                │
│                                         │
│     @property                           │
│     def computed(self):       ← smart   │
│         return something                │
│                                         │
│     @classmethod                        │
│     def from_x(cls, ...):    ← alt init │
│         return cls(...)                 │
│                                         │
│     @staticmethod                       │
│     def utility(...):        ← helper   │
│         return result                   │
│                                         │
│     def __str__(self):       ← print()  │
│     def __repr__(self):      ← debug    │
│     def __eq__(self, other): ← ==       │
│     def __lt__(self, other): ← sorted() │
│     def __len__(self):       ← len()    │
└─────────────────────────────────────────┘

RELATIONSHIPS:
  Inheritance (IS-A):  class Child(Parent):     → when B is a type of A
  Composition (HAS-A): self.parts = [Part()]    → when A contains B
  Prefer composition over inheritance!

KEY RULES:
  1. self = "this specific object"
  2. Always call super().__init__() in child classes
  3. Use @property for validation and computed values
  4. Add __str__ and __repr__ to EVERY class
  5. Single underscore = "internal, don't touch from outside"
```

> **INTERVIEW PREP:**
> - "Explain OOP in your own words" → organizing code by grouping data + behavior
> - "What is self?" → reference to the current object instance
> - "Inheritance vs composition?" → is-a vs has-a, prefer composition
> - "What is __init__?" → constructor, runs when creating an object
> - "What are dunder methods?" → special methods that integrate with Python syntax
> - "What is @property?" → attribute with getter/setter logic behind it
>
> Next: File 10 — Python Error Handling (Week 15, May 24-28)
> Topics: Exceptions, try/except, Custom Exceptions, File I/O, Context Managers
