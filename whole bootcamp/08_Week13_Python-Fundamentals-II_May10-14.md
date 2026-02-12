# PYTHON TRACK — FILE 08 of 48
## PYTHON FUNDAMENTALS II
### Lists, Tuples, Dictionaries, Sets, Comprehensions
**Week 13 | May 10 - May 14, 2025 | 5 Days | 1.5 hrs/day**

---

> **BUILDING ON FILE 07**
>
> File 07 gave you: variables, types, strings, control flow, loops.
> File 08 adds: Python's powerful DATA STRUCTURES — the containers
> that hold and organize your data.
>
> As SRE, you'll use these constantly:
> - **Lists** → server names, log lines, metrics over time
> - **Dicts** → config files, JSON responses, key-value mappings
> - **Sets** → unique IPs, deduplication, finding differences
> - **Tuples** → coordinates, immutable configs, function returns
> - **Comprehensions** → one-liner data transformations (Python superpower!)

| Day | Topic | Key Skills |
|-----|-------|-----------|
| 1 | Lists | Create, index, slice, append, remove, sort, iterate |
| 2 | Lists Advanced | Nested lists, list methods, copying, common patterns |
| 3 | Dictionaries | Create, access, iterate, methods, nested dicts |
| 4 | Tuples & Sets | Immutability, unpacking, set operations, dedup |
| 5 | Comprehensions & Capstone | List/dict/set comprehensions, combined exercises |

---

## DAY 1: LISTS — THE WORKHORSE
**May 10, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: create lists, indexing, slicing, basic methods
> 00:20-01:10 Hands-on: list manipulation exercises
> 01:10-01:30 Drills: list challenges

### Creating Lists

```python
# A list is an ORDERED, MUTABLE collection of items
# Lists can hold ANY type, even mixed types

# Empty list:
empty = []
also_empty = list()

# List of strings:
servers = ["web-01", "web-02", "web-03", "db-01"]

# List of numbers:
cpu_values = [45.2, 87.1, 23.4, 92.5, 15.0]

# Mixed types (valid but usually avoid):
mixed = ["web-01", 8080, True, 3.14]

# List from range:
numbers = list(range(1, 11))     # [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# List from string:
chars = list("ERROR")            # ['E', 'R', 'R', 'O', 'R']

# Length:
len(servers)                     # 4
```

### Indexing & Slicing (Same as Strings!)

```python
servers = ["web-01", "web-02", "web-03", "db-01", "cache-01"]

# Indexing:
servers[0]                   # 'web-01' (first)
servers[-1]                  # 'cache-01' (last)
servers[2]                   # 'web-03'

# Slicing [start:stop:step]:
servers[0:3]                 # ['web-01', 'web-02', 'web-03']
servers[:2]                  # ['web-01', 'web-02'] (first two)
servers[2:]                  # ['web-03', 'db-01', 'cache-01'] (from index 2)
servers[-2:]                 # ['db-01', 'cache-01'] (last two)
servers[::2]                 # ['web-01', 'web-03', 'cache-01'] (every other)
servers[::-1]                # reversed list

# UNLIKE strings, lists are MUTABLE — you can change them:
servers[0] = "web-10"        # replace first element
servers[1:3] = ["api-01", "api-02"]  # replace a slice
```

### Adding & Removing Elements

```python
servers = ["web-01", "web-02"]

# Add to end:
servers.append("web-03")            # ['web-01', 'web-02', 'web-03']

# Add multiple to end:
servers.extend(["db-01", "db-02"])   # [..., 'db-01', 'db-02']
# Same as: servers += ["db-01", "db-02"]

# Insert at specific position:
servers.insert(0, "lb-01")          # insert at beginning

# Remove by value (first occurrence):
servers.remove("db-02")             # removes 'db-02'
# Raises ValueError if not found!

# Remove by index:
last = servers.pop()                # removes AND returns last item
second = servers.pop(1)             # removes AND returns item at index 1

# Remove by index (no return):
del servers[0]                      # removes first item

# Clear everything:
servers.clear()                     # []

# Check membership:
"web-01" in servers                 # True or False
"web-99" not in servers             # True
```

### Sorting & Searching

```python
numbers = [42, 17, 8, 99, 23, 56]

# Sort in place (modifies original):
numbers.sort()                      # [8, 17, 23, 42, 56, 99]
numbers.sort(reverse=True)          # [99, 56, 42, 23, 17, 8]

# Return sorted copy (original unchanged):
original = [42, 17, 8, 99]
ordered = sorted(original)          # [8, 17, 42, 99]
print(original)                     # [42, 17, 8, 99] — unchanged!

# Sort strings:
hosts = ["web-03", "web-01", "web-02"]
hosts.sort()                        # ['web-01', 'web-02', 'web-03']

# Find:
numbers = [10, 20, 30, 20, 40]
numbers.index(20)                   # 1 (first occurrence)
numbers.count(20)                   # 2 (how many times)

# Min, max, sum:
min(numbers)                        # 10
max(numbers)                        # 40
sum(numbers)                        # 120

# Reverse in place:
numbers.reverse()                   # [40, 20, 30, 20, 10]
```

> **SRE PATTERN: COLLECTING METRICS**
>
> ```python
> # Collect response times, then analyze
> response_times = []
> for i in range(100):
>     # Imagine measuring API response here
>     rt = 150 + (i % 30)           # simulated data
>     response_times.append(rt)
>
> print(f"Count: {len(response_times)}")
> print(f"Min:   {min(response_times)}ms")
> print(f"Max:   {max(response_times)}ms")
> print(f"Avg:   {sum(response_times) / len(response_times):.1f}ms")
> ```

### DAY 1 EXERCISES

> **EXERCISE 1: Server Inventory**
>
> a) Create a list of 6 server names (mix of web, db, cache)
> b) Print the first server and last server
> c) Print the middle two servers using slicing
> d) Add a new server "monitor-01" to the end
> e) Insert "lb-01" at the beginning
> f) Remove the 3rd server
> g) Sort the list alphabetically, print it
> h) Reverse the sorted list, print it

> **EXERCISE 2: CPU Metrics**
>
> Given: `cpu_readings = [45, 78, 92, 34, 88, 67, 95, 23, 81, 56]`
> a) Print min, max, average CPU
> b) Count how many readings are above 80
> c) Find the index of the maximum reading
> d) Sort a COPY of the list (keep original intact), print both
> e) Get the top 3 highest readings using sort + slice
> f) Remove all readings below 30 (loop and build new list)

> **EXERCISE 3: List Operations**
>
> a) Create a list of numbers 1-20 using `list(range(...))`
> b) Get only even numbers using slicing (start at index 1, step 2)
> c) Reverse the list without using .reverse() (use slicing)
> d) Combine two lists: [1,2,3] and [4,5,6] using +
> e) Check if 15 is in the list using `in`
> f) Create a list of 10 zeros: `[0] * 10`

> **DRILL: 10 List Challenges (2 min each)**
>
> 1. Create a list of 5 fruits, print the 3rd one
> 2. Append "mango" to your fruit list
> 3. Remove the first fruit from the list
> 4. Sort the list, then reverse it
> 5. Check if "banana" is in the list
> 6. Get the length of the list
> 7. Create a list from the string "hello" (each char separate)
> 8. Sum all numbers in [10, 20, 30, 40, 50]
> 9. Combine [1,2] + [3,4] + [5,6]
> 10. What does `[1,2,3] * 3` produce? Try it.

---

## DAY 2: LISTS ADVANCED
**May 11, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: nested lists, copying, looping patterns, common pitfalls
> 00:20-01:10 Hands-on: advanced list manipulation
> 01:10-01:30 Drills: real-world list problems

### Nested Lists (Lists Inside Lists)

```python
# A list can contain other lists — like a 2D table

# Server matrix: [name, ip, port, status]
servers = [
    ["web-01", "10.0.1.1", 8080, "healthy"],
    ["web-02", "10.0.1.2", 8080, "degraded"],
    ["db-01",  "10.0.2.1", 5432, "healthy"],
    ["cache-01", "10.0.3.1", 6379, "healthy"],
]

# Access nested elements:
servers[0]                   # ['web-01', '10.0.1.1', 8080, 'healthy']
servers[0][0]                # 'web-01'
servers[1][3]                # 'degraded'
servers[2][2]                # 5432

# Loop through nested lists:
print(f"{'Name':<12} {'IP':<15} {'Port':>5} {'Status':<10}")
print("-" * 45)
for server in servers:
    name, ip, port, status = server      # unpacking!
    print(f"{name:<12} {ip:<15} {port:>5} {status:<10}")

# Alternative unpacking directly in the for loop:
for name, ip, port, status in servers:
    if status != "healthy":
        print(f"ALERT: {name} is {status}")
```

### Copying Lists — THE TRAP

```python
# THIS IS A COMMON BUG. Learn it now.

# WRONG — this creates a REFERENCE, not a copy:
original = [1, 2, 3]
wrong_copy = original        # both point to the SAME list!
wrong_copy.append(4)
print(original)              # [1, 2, 3, 4]  ← ORIGINAL CHANGED!

# RIGHT — actual copy:
original = [1, 2, 3]

# Method 1: slice
copy1 = original[:]

# Method 2: list()
copy2 = list(original)

# Method 3: .copy()
copy3 = original.copy()

# Now modifying copy doesn't affect original:
copy1.append(99)
print(original)              # [1, 2, 3] ← safe!

# DEEP COPY for nested lists:
import copy
nested = [[1, 2], [3, 4]]
shallow = nested[:]          # inner lists are still shared!
deep = copy.deepcopy(nested) # completely independent
```

> **WHY THIS MATTERS FOR SRE**
>
> You'll often pass lists to functions or store them in variables.
> If you forget to copy, you'll get mysterious bugs where changing
> one variable affects another. This is a classic Python interview question.

### Useful Looping Patterns

```python
# Pattern 1: Build a new list from an old one (filter)
cpu_values = [45, 87, 23, 92, 15, 88, 95]
high_cpu = []
for cpu in cpu_values:
    if cpu > 80:
        high_cpu.append(cpu)
print(high_cpu)              # [87, 92, 88, 95]

# Pattern 2: Transform each element
servers = ["web-01", "web-02", "db-01"]
upper_servers = []
for s in servers:
    upper_servers.append(s.upper())
# ['WEB-01', 'WEB-02', 'DB-01']

# Pattern 3: Enumerate with index
for i, server in enumerate(servers):
    print(f"#{i+1}: {server}")

# Pattern 4: Zip two lists together
names = ["web-01", "web-02", "db-01"]
cpus  = [45.2, 87.1, 23.4]
for name, cpu in zip(names, cpus):
    print(f"{name}: {cpu}%")
# web-01: 45.2%
# web-02: 87.1%
# db-01: 23.4%

# Pattern 5: Find something
servers = ["web-01", "web-02", "db-01", "cache-01"]
db_server = None
for s in servers:
    if s.startswith("db"):
        db_server = s
        break
print(f"Found: {db_server}")   # Found: db-01

# Pattern 6: Flatten nested list
nested = [[1, 2], [3, 4], [5, 6]]
flat = []
for sublist in nested:
    for item in sublist:
        flat.append(item)
print(flat)                  # [1, 2, 3, 4, 5, 6]
```

### String ↔ List Conversions

```python
# Split string into list:
log_line = "2025-05-11 ERROR web-01 timeout"
parts = log_line.split()         # ['2025-05-11', 'ERROR', 'web-01', 'timeout']

csv = "web-01,10.0.1.1,8080"
fields = csv.split(",")         # ['web-01', '10.0.1.1', '8080']

# Join list into string:
servers = ["web-01", "web-02", "web-03"]
result = ", ".join(servers)      # 'web-01, web-02, web-03'
command = " | ".join(["grep ERROR", "wc -l"])  # 'grep ERROR | wc -l'

# Read lines from a multi-line string:
text = """web-01
web-02
web-03"""
lines = text.strip().splitlines()   # ['web-01', 'web-02', 'web-03']
```

### DAY 2 EXERCISES

> **EXERCISE 1: Server Data Table**
>
> Create a nested list with 5 servers: [name, ip, cpu, memory, status]
> a) Print a formatted table with headers
> b) Find all servers where CPU > 80
> c) Find the server with the highest memory usage
> d) Count servers by status (healthy vs degraded vs down)
> e) Sort by CPU usage (highest first) — hint: `sorted(servers, key=lambda x: x[2], reverse=True)`

> **EXERCISE 2: Log Line Processor**
>
> Given a list of log line strings (space-separated: date time level host message):
> a) Split each line into parts
> b) Extract all unique log levels
> c) Group log lines by level (create separate lists for ERROR, WARN, INFO)
> d) Find the host that appears most often in ERROR lines

> **EXERCISE 3: List Transformation**
>
> a) Given `[1,2,3,4,5]` → create `[2,4,6,8,10]` (multiply each by 2)
> b) Given `["web-01","db-01","web-02","cache-01"]` → extract only web servers
> c) Given two lists of equal length, create pairs using zip: `[(a1,b1), (a2,b2)]`
> d) Flatten `[[1,2],[3,4],[5,6]]` into `[1,2,3,4,5,6]`
> e) Remove duplicates from `[1,3,2,3,1,4,2,5]` while preserving order

> **DRILL: 10 Advanced List Challenges (3 min each)**
>
> 1. Copy a list correctly (not by reference)
> 2. Find the second largest number in a list
> 3. Reverse a list without .reverse() or [::-1] (use a loop)
> 4. Merge two sorted lists into one sorted list
> 5. Count how many strings in a list have length > 5
> 6. Remove all occurrences of a value from a list
> 7. Check if two lists have any common elements
> 8. Split a list into chunks of size 3: [1,2,3,4,5,6] → [[1,2,3],[4,5,6]]
> 9. Rotate a list left by 2: [1,2,3,4,5] → [3,4,5,1,2]
> 10. Given a list of numbers, separate into positive and negative lists

---

## DAY 3: DICTIONARIES — KEY-VALUE POWER
**May 12, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: create dicts, access, iterate, methods, nesting
> 00:20-01:10 Hands-on: dictionary exercises
> 01:10-01:30 Drills: dict challenges

### Creating Dictionaries

```python
# A dict maps KEYS to VALUES. Keys must be unique and immutable.
# Dicts are UNORDERED (well, insertion-ordered since Python 3.7)

# Empty dict:
empty = {}
also_empty = dict()

# Server info:
server = {
    "name": "web-01",
    "ip": "10.0.1.1",
    "port": 8080,
    "healthy": True,
    "tags": ["production", "us-east"],
}

# From list of pairs:
pairs = [("name", "web-01"), ("ip", "10.0.1.1")]
server2 = dict(pairs)

# Using dict() with keyword arguments:
server3 = dict(name="web-01", ip="10.0.1.1", port=8080)
```

### Accessing Values

```python
server = {"name": "web-01", "ip": "10.0.1.1", "port": 8080}

# Square bracket access (raises KeyError if missing):
server["name"]                      # 'web-01'
server["port"]                      # 8080
# server["status"]                  # KeyError! Key doesn't exist

# .get() — safe access (returns None or default if missing):
server.get("name")                  # 'web-01'
server.get("status")                # None (no error!)
server.get("status", "unknown")     # 'unknown' (custom default)

# RULE: Use .get() when the key MIGHT not exist
# Use [] when you KNOW it exists and want an error if it doesn't
```

### Modifying Dictionaries

```python
server = {"name": "web-01", "ip": "10.0.1.1"}

# Add or update:
server["port"] = 8080               # add new key
server["ip"] = "10.0.1.100"         # update existing key

# Update multiple at once:
server.update({"status": "healthy", "cpu": 45.2})

# Remove:
port = server.pop("port")           # removes AND returns value
server.pop("missing", None)         # returns None if missing (no error)
del server["cpu"]                   # remove (raises KeyError if missing)
last = server.popitem()             # removes and returns last pair

# Check if key exists:
"name" in server                    # True
"port" not in server                # True (we removed it)

# Get all keys, values, pairs:
server.keys()                       # dict_keys(['name', 'ip', 'status'])
server.values()                     # dict_values(['web-01', '10.0.1.100', 'healthy'])
server.items()                      # dict_items([('name', 'web-01'), ...])
```

### Iterating Over Dictionaries

```python
server = {"name": "web-01", "ip": "10.0.1.1", "port": 8080, "status": "healthy"}

# Loop over keys (default):
for key in server:
    print(key)               # name, ip, port, status

# Loop over values:
for value in server.values():
    print(value)

# Loop over key-value pairs (MOST COMMON):
for key, value in server.items():
    print(f"{key}: {value}")

# SRE example: print config
config = {
    "max_connections": 100,
    "timeout_seconds": 30,
    "retry_count": 3,
    "log_level": "INFO",
}
print("=== Configuration ===")
for setting, value in config.items():
    print(f"  {setting:<20} = {value}")
```

### Nested Dictionaries

```python
# Dicts inside dicts — how real config/JSON data looks

infrastructure = {
    "web-01": {
        "ip": "10.0.1.1",
        "cpu": 45.2,
        "memory": 62.1,
        "services": ["nginx", "app"],
    },
    "db-01": {
        "ip": "10.0.2.1",
        "cpu": 78.5,
        "memory": 88.3,
        "services": ["postgresql"],
    },
    "cache-01": {
        "ip": "10.0.3.1",
        "cpu": 12.0,
        "memory": 45.0,
        "services": ["redis"],
    },
}

# Access nested values:
infrastructure["web-01"]["ip"]            # '10.0.1.1'
infrastructure["db-01"]["cpu"]            # 78.5
infrastructure["cache-01"]["services"][0] # 'redis'

# Loop through all servers:
for hostname, info in infrastructure.items():
    cpu = info["cpu"]
    status = "ALERT" if cpu > 80 else "OK"
    print(f"{hostname}: CPU={cpu}% [{status}]")

# Safely access nested values:
infrastructure.get("missing-server", {}).get("cpu", "N/A")   # 'N/A'
```

> **SRE RELEVANCE: JSON IS JUST NESTED DICTS**
>
> Every API response, every config file (JSON/YAML), every monitoring
> alert is a nested dictionary. Master dicts = master SRE data handling.
>
> ```python
> # This is basically what a Kubernetes pod status looks like:
> pod = {
>     "metadata": {"name": "api-pod-abc123", "namespace": "production"},
>     "status": {"phase": "Running", "containerStatuses": [
>         {"ready": True, "restartCount": 0}
>     ]}
> }
> pod["status"]["phase"]                    # 'Running'
> pod["status"]["containerStatuses"][0]["restartCount"]  # 0
> ```

### DAY 3 EXERCISES

> **EXERCISE 1: Server Config Manager**
>
> a) Create a dict for a server: name, ip, port, environment, status, tags (list)
> b) Print all config using a for loop over .items()
> c) Update the port to 9090
> d) Add a new key "last_checked" with current date string
> e) Remove the "tags" key safely (use .pop with default)
> f) Check if "cpu" key exists. If not, add it with value 0.0

> **EXERCISE 2: Inventory System**
>
> Create a nested dict of 4 servers (like the infrastructure example).
> a) Print a formatted table of all servers with their CPU and memory
> b) Find the server with the highest CPU usage
> c) Find all servers running "nginx" service
> d) Calculate average memory across all servers
> e) Add a new server to the inventory

> **EXERCISE 3: Word Frequency Counter**
>
> Given a string: "error error warning info error info info debug warning error"
> a) Split into words
> b) Count the frequency of each word using a dict
> c) Print each word and its count
> d) Find the most common word
> e) Sort by frequency (descending)

> **EXERCISE 4: Config File Parser**
>
> Given lines of "key=value" format:
> ```python
> config_lines = [
>     "hostname=web-01",
>     "port=8080",
>     "environment=production",
>     "debug=false",
>     "max_connections=100",
> ]
> ```
> a) Parse into a dictionary
> b) Print all config settings
> c) Convert "port" and "max_connections" to integers
> d) Convert "debug" to boolean

> **DRILL: 10 Dict Challenges (2 min each)**
>
> 1. Create a dict with 3 key-value pairs, print all keys
> 2. Access a value using .get() with a default
> 3. Add a new key-value pair to an existing dict
> 4. Loop through a dict printing "key -> value"
> 5. Merge two dicts using .update()
> 6. Check if a key exists using `in`
> 7. Get the number of keys in a dict using len()
> 8. Create a dict from two lists using zip: keys=["a","b"], values=[1,2]
> 9. Remove a key and get its value using .pop()
> 10. Swap keys and values in a dict: {"a": 1} → {1: "a"}

---

## DAY 4: TUPLES & SETS
**May 13, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: tuples (immutability, unpacking), sets (uniqueness, operations)
> 00:20-01:10 Hands-on: tuple and set exercises
> 01:10-01:30 Drills: combined challenges

### Tuples — Immutable Lists

```python
# Tuples are like lists but CANNOT be changed after creation
# Use them for data that shouldn't change

# Create:
point = (10, 20)
server_info = ("web-01", "10.0.1.1", 8080)
single = (42,)                    # note the comma! (42) is just an int
empty = ()

# Access (same as lists):
server_info[0]                    # 'web-01'
server_info[-1]                   # 8080
server_info[0:2]                  # ('web-01', '10.0.1.1')

# CANNOT modify:
# server_info[0] = "web-02"      # TypeError: tuple doesn't support assignment

# Tuple UNPACKING — super useful!
name, ip, port = server_info
print(name)                       # 'web-01'
print(port)                       # 8080

# Swap variables (Python trick using tuples):
a, b = 1, 2
a, b = b, a                      # now a=2, b=1

# Ignore values with underscore:
name, _, port = server_info       # ignore ip
first, *rest = [1, 2, 3, 4, 5]   # first=1, rest=[2,3,4,5]
first, *middle, last = [1,2,3,4,5]  # first=1, middle=[2,3,4], last=5

# Functions returning multiple values use tuples:
def get_server_stats():
    return "web-01", 87.5, 64.2    # returns a tuple

name, cpu, memory = get_server_stats()
```

> **WHEN TO USE TUPLE vs LIST**
>
> - **Tuple**: data that shouldn't change (coordinates, config constants, 
>   function returns, dict keys)
> - **List**: data that grows/shrinks/changes (server inventory, log lines,
>   metrics over time)
>
> Tuples are also slightly faster and use less memory than lists.

### Sets — Unique Collections

```python
# A set is an UNORDERED collection of UNIQUE elements
# No duplicates allowed!

# Create:
ips = {"10.0.1.1", "10.0.1.2", "10.0.1.3"}
empty_set = set()          # NOT {} — that creates an empty dict!

# From list (removes duplicates):
log_levels = ["ERROR", "INFO", "ERROR", "WARN", "INFO", "INFO"]
unique_levels = set(log_levels)
print(unique_levels)       # {'ERROR', 'INFO', 'WARN'} — duplicates gone!

# Add and remove:
ips.add("10.0.1.4")       # add one
ips.update({"10.0.1.5", "10.0.1.6"})  # add multiple
ips.discard("10.0.1.6")   # remove (no error if missing)
ips.remove("10.0.1.5")    # remove (KeyError if missing)

# Membership test (VERY fast — O(1)):
"10.0.1.1" in ips          # True — much faster than list!

# Length:
len(ips)                   # number of unique elements
```

### Set Operations — THE KILLER FEATURE

```python
# These are incredibly useful for SRE work

prod_servers = {"web-01", "web-02", "web-03", "db-01"}
monitored = {"web-01", "web-02", "db-01", "cache-01"}

# INTERSECTION — servers in BOTH sets:
both = prod_servers & monitored
# or: prod_servers.intersection(monitored)
# {'web-01', 'web-02', 'db-01'}

# UNION — all servers from either set:
all_servers = prod_servers | monitored
# or: prod_servers.union(monitored)
# {'web-01', 'web-02', 'web-03', 'db-01', 'cache-01'}

# DIFFERENCE — in prod but NOT monitored:
unmonitored = prod_servers - monitored
# or: prod_servers.difference(monitored)
# {'web-03'}  ← THIS SERVER HAS NO MONITORING! Alert!

# SYMMETRIC DIFFERENCE — in one but not both:
mismatch = prod_servers ^ monitored
# {'web-03', 'cache-01'}

# Subset/superset:
small = {"web-01", "web-02"}
small.issubset(prod_servers)        # True — small is inside prod
prod_servers.issuperset(small)      # True
```

> **SRE USE CASE: FINDING GAPS**
>
> ```python
> # Find unmonitored production servers:
> prod = {"web-01", "web-02", "web-03", "db-01", "api-01"}
> monitored = {"web-01", "web-02", "db-01"}
> 
> gaps = prod - monitored
> if gaps:
>     print(f"ALERT: {len(gaps)} servers without monitoring!")
>     for server in sorted(gaps):
>         print(f"  - {server}")
> # ALERT: 2 servers without monitoring!
> #   - api-01
> #   - web-03
> ```

### DAY 4 EXERCISES

> **EXERCISE 1: Tuple Practice**
>
> a) Create a tuple of (hostname, ip, port, status) for 3 servers
> b) Unpack each into separate variables
> c) Create a list of tuples, loop through and print each server
> d) Try to change a tuple element — observe the error
> e) Use `*rest` unpacking: `first, *remaining = (1, 2, 3, 4, 5)`
> f) Swap two variables using tuple unpacking

> **EXERCISE 2: Set Operations for SRE**
>
> Given:
> ```python
> prod_hosts = {"web-01", "web-02", "web-03", "api-01", "db-01", "db-02"}
> staging_hosts = {"web-01", "api-01", "db-01"}
> monitored_hosts = {"web-01", "web-02", "api-01", "db-01", "cache-01"}
> patched_hosts = {"web-01", "web-02", "web-03", "db-01"}
> ```
> a) Which prod hosts are NOT monitored?
> b) Which monitored hosts are NOT in prod? (might be decommissioned)
> c) Which prod hosts are NOT patched? (security risk!)
> d) Which hosts exist in ALL three sets (prod, staging, monitored)?
> e) Total unique hosts across all sets?

> **EXERCISE 3: Deduplicate & Analyze**
>
> Given: `ips = ["10.0.1.1", "10.0.1.2", "10.0.1.1", "10.0.1.3", "10.0.1.2", "10.0.1.1", "10.0.1.4"]`
> a) How many total entries? How many unique?
> b) Convert to set, back to sorted list
> c) Find which IPs appear more than once (use count from original list)
> d) Given another list of blocked IPs, find which of our IPs are blocked

> **DRILL: 10 Tuple & Set Challenges (2 min each)**
>
> 1. Create a tuple of 5 numbers, get the sum
> 2. Unpack (x, y, z) = (10, 20, 30), print y
> 3. Create a set from [1,1,2,2,3,3], print it
> 4. Add an element to a set
> 5. Find the intersection of {1,2,3} and {2,3,4}
> 6. Find elements in {1,2,3} but not in {2,3,4}
> 7. Check if {1,2} is a subset of {1,2,3,4}
> 8. Remove duplicates from a list using set(), convert back to list
> 9. Can a tuple be a dict key? Can a list? Test both.
> 10. Create a set of all unique characters in "mississippi"

---

## DAY 5: COMPREHENSIONS & CAPSTONE
**May 14, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: list, dict, set comprehensions
> 00:20-01:10 Hands-on: comprehension exercises + capstone
> 01:10-01:30 Drills: combined data structure challenges

### List Comprehensions — Python's Superpower

```python
# A comprehension creates a new list by transforming/filtering another

# Basic syntax: [expression for item in iterable]

# Without comprehension (old way):
squares = []
for n in range(10):
    squares.append(n ** 2)

# With comprehension (Python way):
squares = [n ** 2 for n in range(10)]
# [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

# With filter: [expression for item in iterable if condition]
high_cpu = [cpu for cpu in [45, 87, 23, 92, 15] if cpu > 80]
# [87, 92]

# Transform + filter:
servers = ["web-01", "db-01", "web-02", "cache-01", "web-03"]
web_servers = [s.upper() for s in servers if s.startswith("web")]
# ['WEB-01', 'WEB-02', 'WEB-03']

# Nested comprehension (flatten):
matrix = [[1,2,3], [4,5,6], [7,8,9]]
flat = [num for row in matrix for num in row]
# [1, 2, 3, 4, 5, 6, 7, 8, 9]

# With if-else (ternary inside):
statuses = [("OK" if cpu < 80 else "ALERT") for cpu in [45, 87, 23, 92]]
# ['OK', 'ALERT', 'OK', 'ALERT']
```

### Dict Comprehensions

```python
# {key_expr: value_expr for item in iterable}

# Create dict from two lists:
names = ["web-01", "web-02", "db-01"]
cpus = [45.2, 87.1, 23.4]
server_cpu = {name: cpu for name, cpu in zip(names, cpus)}
# {'web-01': 45.2, 'web-02': 87.1, 'db-01': 23.4}

# Filter:
high_cpu = {name: cpu for name, cpu in server_cpu.items() if cpu > 80}
# {'web-02': 87.1}

# Transform:
# Swap keys and values:
cpu_server = {cpu: name for name, cpu in server_cpu.items()}

# Count word frequency:
words = "error error warning info error warning".split()
freq = {}
for w in words:
    freq[w] = freq.get(w, 0) + 1
# Or with comprehension + count:
freq = {w: words.count(w) for w in set(words)}
# {'error': 3, 'warning': 2, 'info': 1}
```

### Set Comprehensions

```python
# {expression for item in iterable}

# Unique first letters:
servers = ["web-01", "web-02", "db-01", "cache-01", "web-03"]
prefixes = {s.split("-")[0] for s in servers}
# {'web', 'db', 'cache'}

# Unique log levels:
logs = ["ERROR: timeout", "INFO: ok", "ERROR: disk", "WARN: high cpu"]
levels = {line.split(":")[0] for line in logs}
# {'ERROR', 'INFO', 'WARN'}
```

> **WHEN TO USE COMPREHENSIONS**
>
> DO use: simple transforms, filters, one-liners that are readable
> DON'T use: complex logic, side effects, multiple conditions
>
> If a comprehension is hard to read, use a regular for loop instead.
> Readability > cleverness. Always.

### Week Capstone: Infrastructure Dashboard

```python
# Combine EVERYTHING from this week!

# --- Infrastructure Data ---
infrastructure = {
    "web-01":   {"ip": "10.0.1.1",  "cpu": 45.2, "mem": 62.1, "env": "prod",    "services": ["nginx", "app"]},
    "web-02":   {"ip": "10.0.1.2",  "cpu": 87.1, "mem": 71.5, "env": "prod",    "services": ["nginx", "app"]},
    "web-03":   {"ip": "10.0.1.3",  "cpu": 23.4, "mem": 45.0, "env": "staging", "services": ["nginx", "app"]},
    "db-01":    {"ip": "10.0.2.1",  "cpu": 92.5, "mem": 88.3, "env": "prod",    "services": ["postgresql"]},
    "db-02":    {"ip": "10.0.2.2",  "cpu": 34.1, "mem": 55.0, "env": "staging", "services": ["postgresql"]},
    "cache-01": {"ip": "10.0.3.1",  "cpu": 15.0, "mem": 78.9, "env": "prod",    "services": ["redis"]},
}

# --- Dashboard Header ---
print("=" * 65)
print(f"{'INFRASTRUCTURE DASHBOARD':^65}")
print("=" * 65)

# --- Server Table ---
print(f"\n{'Host':<12} {'IP':<14} {'CPU':>5} {'Mem':>5} {'Env':<8} {'Status'}")
print("-" * 65)

for host, info in infrastructure.items():
    cpu = info["cpu"]
    if cpu > 90:
        status = "CRITICAL"
    elif cpu > 80:
        status = "WARNING"
    else:
        status = "OK"
    print(f"{host:<12} {info['ip']:<14} {cpu:>4.1f}% {info['mem']:>4.1f}% {info['env']:<8} {status}")

# --- Analysis using comprehensions ---
print("\n--- Analysis ---")

# Prod servers only:
prod_hosts = [h for h, i in infrastructure.items() if i["env"] == "prod"]
print(f"Production servers: {', '.join(prod_hosts)}")

# Average CPU:
all_cpus = [i["cpu"] for i in infrastructure.values()]
avg_cpu = sum(all_cpus) / len(all_cpus)
print(f"Average CPU: {avg_cpu:.1f}%")

# Alerts:
alerts = {h: i["cpu"] for h, i in infrastructure.items() if i["cpu"] > 80}
if alerts:
    print(f"\nALERTS ({len(alerts)} servers):")
    for host, cpu in sorted(alerts.items(), key=lambda x: x[1], reverse=True):
        print(f"  {host}: CPU at {cpu}%")

# Unique services:
all_services = set()
for info in infrastructure.values():
    all_services.update(info["services"])
print(f"\nServices running: {', '.join(sorted(all_services))}")

# Hosts per environment:
envs = {i["env"] for i in infrastructure.values()}
for env in sorted(envs):
    count = sum(1 for i in infrastructure.values() if i["env"] == env)
    print(f"  {env}: {count} servers")
```

### DAY 5 EXERCISES

> **EXERCISE 1: Comprehension Practice**
>
> a) Create a list of squares from 1-20 using comprehension
> b) Filter: only even squares from the list above
> c) From server names list, extract only those containing "web"
> d) Create a dict mapping numbers 1-10 to their cubes: {1: 1, 2: 8, ...}
> e) From a list of strings, create a set of their lengths

> **EXERCISE 2: Data Processing Pipeline**
>
> Given:
> ```python
> raw_data = [
>     "web-01,45.2,62.1,healthy",
>     "web-02,87.1,71.5,degraded",
>     "db-01,92.5,88.3,critical",
>     "cache-01,15.0,78.9,healthy",
> ]
> ```
> a) Parse each line into a dict (split by comma, assign keys)
> b) Store all dicts in a list
> c) Use comprehension to find all servers with CPU > 80
> d) Calculate average CPU using comprehension + sum/len
> e) Create a dict mapping hostname → status

> **EXERCISE 3: Full Dashboard (Capstone)**
>
> Build your own dashboard that:
> a) Defines infrastructure data as nested dicts (at least 6 servers)
> b) Prints a formatted table with alignment
> c) Shows alerts for any metric > 80%
> d) Shows summary: total servers, per-environment count, avg metrics
> e) Uses at least 3 comprehensions
> f) Uses at least 1 set operation

> **DRILL: 10 Comprehension Challenges (3 min each)**
>
> 1. `[x for x in range(20) if x % 3 == 0]` — what does this produce?
> 2. Convert ["hello", "world"] to ["HELLO", "WORLD"] with comprehension
> 3. Create a dict of {word: len(word)} from a sentence
> 4. Flatten [[1,2],[3,4],[5,6]] with a nested comprehension
> 5. Get unique first characters from a list of words (set comprehension)
> 6. Create a list of (index, value) tuples from a list using enumerate
> 7. Filter a dict: keep only entries where value > 50
> 8. Create a list of "even" or "odd" for numbers 1-10
> 9. From a dict of server:cpu, get the server with max CPU
> 10. `{i: i**2 for i in range(5)}` — what dict does this create?

---

## WEEK 13 SUMMARY

> **FILE 08 COMPLETE — PYTHON FUNDAMENTALS II**
>
> ✓ Lists: create, index, slice, append, remove, sort, iterate
> ✓ Lists advanced: nested lists, copying (the trap!), zip, enumerate
> ✓ Dictionaries: create, access (.get!), iterate, nested dicts
> ✓ Tuples: immutability, unpacking, multiple returns
> ✓ Sets: uniqueness, intersection, union, difference (gap analysis!)
> ✓ Comprehensions: list, dict, set — Python's one-liner superpower
> ✓ 45+ exercises with SRE context throughout
>
> **INTERVIEW PREP:**
> - "What's the difference between a list and tuple?" (mutable vs immutable)
> - "How do you remove duplicates from a list?" (`list(set(my_list))`)
> - "What happens when you copy a list with `=`?" (reference, not copy!)
> - "What are dict comprehensions?" (one-liner dict creation)
> - "How do you safely access a dict key?" (`.get(key, default)`)
>
> Next: File 09 — Python OOP (Week 14, May 17-21)
> Topics: Classes, Objects, __init__, Methods, Inheritance, Dunder Methods
