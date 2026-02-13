# PYTHON TRACK — FILE 11 of 48
## PYTHON ADVANCED I
### Decorators, Generators, Iterators, Advanced Context Managers, functools
**Week 16 | May 31 - Jun 4, 2025 | 5 Days | 1.5 hrs/day**

---

> **WHY THESE TOPICS MATTER**
>
> Files 07-10 gave you Python fundamentals. Now we level up.
>
> These topics appear in two places:
> 1. **Interviews** — "Explain decorators" is a top Python interview question
> 2. **Real SRE code** — every monitoring library, every CLI tool, every
>    framework you'll use (FastAPI, Click, pytest) is BUILT on these concepts
>
> The goal: understand these well enough to USE them and EXPLAIN them.
> You don't need to invent new ones from scratch — you need to recognize
> them and know what they're doing in code you read.

| Day | Topic | Key Skills |
|-----|-------|-----------|
| 1 | Closures & Decorators Basics | First-class functions, closures, simple decorators |
| 2 | Decorators Deep Dive | Args, stacking, functools.wraps, real-world decorators |
| 3 | Iterators & Generators | __iter__, __next__, yield, generator expressions |
| 4 | Advanced Generators | yield from, send, pipelines, lazy processing |
| 5 | Context Managers Advanced & Capstone | Class-based, contextlib, combined patterns |

---

## DAY 1: CLOSURES & DECORATOR BASICS
**May 31, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: functions as objects, closures, basic decorators
> 00:20-01:10 Hands-on: write your first decorators
> 01:10-01:30 Drills: decorator exercises

### Functions Are Objects (This Unlocks Everything)

```python
# In Python, functions are OBJECTS — just like strings and lists.
# You can store them in variables, pass them around, return them.

def greet(name):
    return f"Hello, {name}!"

# Store in a variable:
say_hello = greet              # NOT greet() — no parentheses!
print(say_hello("Siddharth"))  # Hello, Siddharth!

# Pass to another function:
def call_twice(func, arg):
    print(func(arg))
    print(func(arg))

call_twice(greet, "World")
# Hello, World!
# Hello, World!

# Store in a list:
operations = [str.upper, str.lower, str.title]
text = "hello world"
for op in operations:
    print(op(text))
# HELLO WORLD
# hello world
# Hello World

# Functions have attributes:
print(greet.__name__)          # 'greet'
print(greet.__doc__)           # None (no docstring)
```

### Functions Inside Functions (Nested Functions)

```python
def outer():
    message = "Hello from outer!"   # local to outer

    def inner():
        print(message)              # inner can ACCESS outer's variables
    
    inner()                         # call inner inside outer

outer()      # Hello from outer!
# inner()   # NameError — inner doesn't exist outside outer
```

### Closures — Functions That Remember

```python
# A closure is a function that REMEMBERS variables from its enclosing scope,
# even after that scope has finished executing.

def make_greeter(greeting):
    """Returns a function that uses 'greeting'."""
    
    def greeter(name):
        return f"{greeting}, {name}!"    # 'greeting' is captured from outer
    
    return greeter                        # return the FUNCTION, not the result

# Create specialized greeters:
hello = make_greeter("Hello")
hi = make_greeter("Hi")
howdy = make_greeter("Howdy")

print(hello("Siddharth"))       # Hello, Siddharth!
print(hi("Siddharth"))          # Hi, Siddharth!
print(howdy("Siddharth"))       # Howdy, Siddharth!

# 'greeting' is stored inside the closure:
print(hello.__closure__[0].cell_contents)    # 'Hello'
```

> **WHY CLOSURES MATTER**
>
> Closures let you create CUSTOMIZED functions at runtime.
> This is the foundation of decorators.
>
> SRE example:
> ```python
> def make_threshold_checker(metric_name, warning, critical):
>     def check(value):
>         if value > critical:
>             return f"CRITICAL: {metric_name}={value}"
>         elif value > warning:
>             return f"WARNING: {metric_name}={value}"
>         return f"OK: {metric_name}={value}"
>     return check
>
> check_cpu = make_threshold_checker("cpu", 80, 90)
> check_mem = make_threshold_checker("memory", 70, 85)
>
> print(check_cpu(95))    # CRITICAL: cpu=95
> print(check_mem(75))    # WARNING: memory=75
> ```

### Your First Decorator

```python
# A decorator is a function that WRAPS another function to add behavior.

# THE PATTERN:
# 1. Takes a function as input
# 2. Defines a wrapper function
# 3. Wrapper calls the original function (plus extra stuff)
# 4. Returns the wrapper

def timer(func):
    """Decorator that measures execution time."""
    import time
    
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)           # call original function
        elapsed = time.time() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result                             # return original result
    
    return wrapper

# Use the decorator:

# Method 1: Manual wrapping
def slow_function():
    import time
    time.sleep(0.5)
    return "done"

slow_function = timer(slow_function)    # wrap it
slow_function()                          # slow_function took 0.5001s

# Method 2: @ syntax (syntactic sugar — same thing!)
@timer
def another_slow_function():
    import time
    time.sleep(0.3)
    return "also done"

another_slow_function()                  # another_slow_function took 0.3001s

# @timer is EXACTLY equivalent to:
# another_slow_function = timer(another_slow_function)
```

### Understanding the @ Syntax

```python
# When Python sees:
@timer
def my_func():
    pass

# It does this AUTOMATICALLY:
# Step 1: Define my_func normally
# Step 2: Call timer(my_func) → returns wrapper function
# Step 3: Assign: my_func = wrapper
# Now 'my_func' is actually the wrapper!

# So when you call my_func(), you're calling wrapper(),
# which calls the original my_func inside it.
```

### DAY 1 EXERCISES

> **EXERCISE 1: Step-by-Step Closure**
>
> a) Write a function `make_multiplier(factor)` that returns a function
>    which multiplies its input by `factor`
> b) Create `double = make_multiplier(2)` and `triple = make_multiplier(3)`
> c) Test: `double(5)` → 10, `triple(5)` → 15
> d) Print `double.__closure__[0].cell_contents` to see the captured value

> **EXERCISE 2: Your First Decorator**
>
> Write a `@logger` decorator that:
> a) Prints "Calling {function_name} with args={args}, kwargs={kwargs}"
> b) Calls the original function
> c) Prints "Returned: {result}"
> d) Returns the result
> e) Test with a function that adds two numbers

> **EXERCISE 3: Counter Decorator**
>
> Write a `@count_calls` decorator that:
> a) Tracks how many times a function is called
> b) Stores the count as an attribute on the wrapper: `wrapper.call_count`
> c) Prints "Call #{n}" each time
> d) Test: call the function 5 times, print `func.call_count`

> **EXERCISE 4: SRE Alert Decorator**
>
> Write a `@alert_on_error` decorator that:
> a) Wraps a function in try/except
> b) If the function raises an exception, print "ALERT: {func_name} failed: {error}"
> c) Re-raises the exception
> d) If successful, just return the result
> e) Test with functions that sometimes raise errors

> **DRILL: 10 Closure/Decorator Challenges (3 min each)**
>
> 1. Store a function in a variable, call it via the variable
> 2. Pass a function as an argument to another function
> 3. Return a function from a function
> 4. Write a closure that captures a list and appends to it
> 5. Write a decorator that prints "Before" and "After" around a function call
> 6. Verify that `@decorator` is the same as `func = decorator(func)`
> 7. Check what `func.__name__` returns after decorating (hint: it's "wrapper")
> 8. Write a decorator that does nothing (just returns the original result)
> 9. Can you decorate a class method? Try it.
> 10. What happens if a decorator doesn't call the original function?

---

## DAY 2: DECORATORS DEEP DIVE
**Jun 1, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: functools.wraps, decorators with args, stacking
> 00:20-01:10 Hands-on: real-world decorators
> 01:10-01:30 Drills: advanced decorator exercises

### The functools.wraps Fix

```python
import functools

# PROBLEM: After decorating, the function loses its identity:
@timer
def my_func():
    """My important function."""
    pass

print(my_func.__name__)     # 'wrapper' ← WRONG! Should be 'my_func'
print(my_func.__doc__)      # None ← WRONG! Should be 'My important function.'

# FIX: Use @functools.wraps(func) inside your decorator:
def timer(func):
    @functools.wraps(func)           # ← preserves __name__, __doc__, etc.
    def wrapper(*args, **kwargs):
        import time
        start = time.time()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.time() - start:.4f}s")
        return result
    return wrapper

@timer
def my_func():
    """My important function."""
    pass

print(my_func.__name__)     # 'my_func' ← correct!
print(my_func.__doc__)      # 'My important function.' ← correct!

# RULE: ALWAYS use @functools.wraps in every decorator you write.
```

### Decorators with Arguments

```python
import functools

# What if you want: @retry(max_attempts=3)?
# You need a decorator FACTORY — a function that RETURNS a decorator.

def retry(max_attempts=3, delay=1):
    """Decorator factory — returns a decorator configured with these args."""
    
    def decorator(func):                         # the actual decorator
        @functools.wraps(func)
        def wrapper(*args, **kwargs):            # the wrapper
            last_exception = None
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_exception = e
                    print(f"Attempt {attempt}/{max_attempts} failed: {e}")
                    if attempt < max_attempts:
                        import time
                        time.sleep(delay)
            raise last_exception
        return wrapper
    return decorator

# Three layers:
# retry(3, 1) → returns decorator
# decorator(func) → returns wrapper
# wrapper(*args) → calls func with retry logic

@retry(max_attempts=5, delay=2)
def call_api():
    import random
    if random.random() < 0.7:
        raise ConnectionError("Timeout")
    return {"status": "ok"}

result = call_api()    # retries up to 5 times
```

### Stacking Decorators

```python
import functools
import time

def timer(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print(f"[TIMER] {func.__name__}: {time.time()-start:.4f}s")
        return result
    return wrapper

def logger(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        print(f"[LOG] Calling {func.__name__}")
        result = func(*args, **kwargs)
        print(f"[LOG] {func.__name__} returned {result}")
        return result
    return wrapper

# Stack decorators — order matters! Bottom decorator wraps first.
@timer           # outer: times the whole thing (including logging)
@logger          # inner: logs the call
def add(a, b):
    return a + b

add(3, 4)
# [LOG] Calling add
# [LOG] add returned 7
# [TIMER] add: 0.0001s

# Execution order: timer wraps logger wraps add
# timer → logger → add → logger → timer
```

### Real-World SRE Decorators

```python
import functools
import time
import logging

logging.basicConfig(level=logging.INFO)
log = logging.getLogger(__name__)

# 1. Rate limiter
def rate_limit(calls_per_second=10):
    """Limit how often a function can be called."""
    min_interval = 1.0 / calls_per_second
    
    def decorator(func):
        last_called = [0.0]      # mutable to allow closure modification
        
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            if elapsed < min_interval:
                wait = min_interval - elapsed
                time.sleep(wait)
            last_called[0] = time.time()
            return func(*args, **kwargs)
        return wrapper
    return decorator

@rate_limit(calls_per_second=5)
def query_api(endpoint):
    return f"Response from {endpoint}"


# 2. Cache/Memoize (functools has this built in!)
@functools.lru_cache(maxsize=128)
def expensive_dns_lookup(hostname):
    """Cache DNS results to avoid repeated lookups."""
    import socket
    return socket.gethostbyname(hostname)

# First call: slow (actual DNS lookup)
# Subsequent calls with same hostname: instant (cached)
expensive_dns_lookup("google.com")    # slow
expensive_dns_lookup("google.com")    # instant (cached!)
print(expensive_dns_lookup.cache_info())  # hits, misses, etc.


# 3. Validate arguments
def validate_port(func):
    """Ensure port argument is valid."""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        port = kwargs.get("port") or (args[1] if len(args) > 1 else None)
        if port is not None and (port < 1 or port > 65535):
            raise ValueError(f"Invalid port: {port}")
        return func(*args, **kwargs)
    return wrapper

@validate_port
def connect(host, port):
    print(f"Connecting to {host}:{port}")
```

### DAY 2 EXERCISES

> **EXERCISE 1: Complete Retry Decorator**
>
> Build `@retry(max_attempts, delay, backoff_factor, exceptions)`:
> a) Retries on failure
> b) Exponential backoff: delay *= backoff_factor each retry
> c) Only catches specific exceptions (e.g., `exceptions=(ConnectionError, TimeoutError)`)
> d) Logs each attempt
> e) Uses @functools.wraps
> f) Test with a function that randomly fails

> **EXERCISE 2: Cache Decorator (Build Your Own)**
>
> Build `@cache` without using functools.lru_cache:
> a) Store results in a dict: {args: result}
> b) If same args seen before, return cached result
> c) Print "CACHE HIT" or "CACHE MISS"
> d) Add a `cache.clear()` method to reset
> e) Test with an expensive function

> **EXERCISE 3: Stacking Practice**
>
> Create 3 decorators: @timer, @logger, @validate_args
> a) Stack all three on one function
> b) Try different orders — see how output changes
> c) Which order makes the most sense for SRE? Why?

> **EXERCISE 4: Decorator with State**
>
> Build `@circuit_breaker(failure_threshold=5, cooldown=30)`:
> a) Tracks consecutive failures
> b) After threshold failures, raises immediately for cooldown period
> c) After cooldown, allows one test call
> d) Resets on success
> e) This is a REAL production pattern!

> **DRILL: 10 Decorator Challenges (3 min each)**
>
> 1. Write a decorator with @functools.wraps — verify __name__ is preserved
> 2. Write a decorator factory: @repeat(n=3) — calls function n times
> 3. Use @functools.lru_cache on a recursive fibonacci function
> 4. Stack two decorators — predict the execution order
> 5. Write a decorator that suppresses exceptions and returns None instead
> 6. Write a decorator that converts the return value to uppercase
> 7. Write a decorator that only allows a function to be called once
> 8. What does `functools.lru_cache.cache_info()` show?
> 9. Can you decorate a class? (yes — `@decorator class Foo: ...`)
> 10. Write a @deprecated decorator that prints a warning

---

## DAY 3: ITERATORS & GENERATORS
**Jun 2, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: iteration protocol, __iter__, __next__, generators
> 00:20-01:10 Hands-on: build iterators and generators
> 01:10-01:30 Drills: iteration challenges

### The Iteration Protocol

```python
# When you write 'for item in something', Python does this:

# 1. Calls iter(something) → gets an ITERATOR object
# 2. Repeatedly calls next(iterator) → gets one item at a time
# 3. When StopIteration is raised → loop ends

# Let's see it manually:
servers = ["web-01", "web-02", "web-03"]

# What 'for' does behind the scenes:
iterator = iter(servers)        # step 1: get iterator
print(next(iterator))           # 'web-01'
print(next(iterator))           # 'web-02'
print(next(iterator))           # 'web-03'
# next(iterator)                # StopIteration!

# This is EXACTLY what 'for server in servers' does.
```

### Building a Custom Iterator (Class-Based)

```python
class Countdown:
    """Counts down from n to 1."""
    
    def __init__(self, start):
        self.start = start
    
    def __iter__(self):
        """Return the iterator object (self)."""
        self.current = self.start
        return self
    
    def __next__(self):
        """Return next value or raise StopIteration."""
        if self.current <= 0:
            raise StopIteration
        value = self.current
        self.current -= 1
        return value

# Use it:
for n in Countdown(5):
    print(n)                     # 5, 4, 3, 2, 1

# Also works with list(), sum(), etc:
print(list(Countdown(3)))       # [3, 2, 1]
```

### Generators — The Easy Way to Make Iterators

```python
# A generator is a function that uses 'yield' instead of 'return'.
# Each yield pauses the function and produces a value.
# Next call resumes from where it paused.

def countdown(start):
    """Generator version — much simpler!"""
    current = start
    while current > 0:
        yield current            # pause here, produce value
        current -= 1             # resume here on next call

# Use exactly like the class version:
for n in countdown(5):
    print(n)                     # 5, 4, 3, 2, 1

# But MUCH simpler to write than the class!
```

### How yield Works (Step by Step)

```python
def simple_gen():
    print("Step 1")
    yield "A"                    # pause, produce "A"
    print("Step 2")
    yield "B"                    # pause, produce "B"
    print("Step 3")
    yield "C"                    # pause, produce "C"
    print("Done")               # runs when exhausted

gen = simple_gen()               # creates generator object (nothing executes yet!)
print(next(gen))                 # Step 1 → 'A'
print(next(gen))                 # Step 2 → 'B'
print(next(gen))                 # Step 3 → 'C'
# next(gen)                      # Done → StopIteration

# KEY INSIGHT: The function's state is FROZEN between yields.
# Local variables, position in code — all preserved.
# This is lazy evaluation — compute values one at a time.
```

### Why Generators Matter — Memory Efficiency

```python
# PROBLEM: Processing a huge log file

# BAD — loads ENTIRE file into memory:
def read_all_errors(filepath):
    lines = open(filepath).readlines()       # 10 GB in memory!
    errors = [l for l in lines if "ERROR" in l]
    return errors

# GOOD — generator processes one line at a time:
def read_errors(filepath):
    with open(filepath) as f:
        for line in f:                       # file is already an iterator!
            if "ERROR" in line:
                yield line.strip()           # produce one at a time

# Usage — only ONE line in memory at a time:
for error in read_errors("/var/log/app.log"):
    print(error)

# Works on a 100 GB file because we never load the whole thing.

# Real numbers:
# List of 10 million items: ~400 MB in memory
# Generator of 10 million items: ~100 bytes (just the generator state!)
```

### Generator Expressions (One-Liner Generators)

```python
# Like list comprehensions but with () instead of []

# List comprehension — creates entire list in memory:
squares_list = [x**2 for x in range(1000000)]     # ~8 MB in memory

# Generator expression — lazy, one at a time:
squares_gen = (x**2 for x in range(1000000))       # ~100 bytes!

# Use gen expressions when you only need to iterate once:
total = sum(x**2 for x in range(1000000))          # no list created!
any_big = any(cpu > 90 for cpu in cpu_readings)    # stops at first True

# SRE example:
log_lines = (line.strip() for line in open("app.log"))
errors = (line for line in log_lines if "ERROR" in line)
first_10_errors = []
for i, error in enumerate(errors):
    if i >= 10:
        break
    first_10_errors.append(error)
# Processed potentially millions of lines, kept only 10 in memory
```

### DAY 3 EXERCISES

> **EXERCISE 1: Custom Iterator — ServerPool**
>
> Build a class `ServerPool` that:
> a) Takes a list of server names
> b) Implements __iter__ and __next__
> c) Cycles through servers infinitely (round-robin)
> d) Test: `pool = ServerPool(["web-01", "web-02", "web-03"])`
>    Get 10 values — should cycle: web-01, web-02, web-03, web-01, ...

> **EXERCISE 2: Generator — Log Reader**
>
> Write generators for:
> a) `read_lines(filepath)` — yields each line stripped
> b) `filter_level(lines, level)` — yields only lines matching level
> c) `parse_entry(lines)` — yields dicts with {date, time, level, message}
> d) Chain them: `parse_entry(filter_level(read_lines("app.log"), "ERROR"))`
> e) This is a GENERATOR PIPELINE — very powerful!

> **EXERCISE 3: Fibonacci Generator**
>
> a) Write a generator `fib()` that yields Fibonacci numbers forever
> b) Use it to get the first 20 Fibonacci numbers
> c) Find the first Fibonacci number greater than 1 million
> d) Compare memory: list of 1M Fibonacci vs generator

> **EXERCISE 4: Generator Expression Practice**
>
> a) Sum of squares from 1-1000 using gen expression
> b) Find if ANY server name starts with "db" (use any())
> c) Count lines in a file without loading it all: `sum(1 for _ in open(f))`
> d) Get first item matching a condition (use next())

> **DRILL: 10 Iterator/Generator Challenges (3 min each)**
>
> 1. Write a generator that yields even numbers from 0 to 20
> 2. Convert a list comprehension to a generator expression
> 3. Write a generator that yields lines from a file
> 4. Use next() to get just the first yielded value
> 5. What does `list(gen)` do? (consumes the entire generator)
> 6. Can you iterate a generator TWICE? (no! it's exhausted)
> 7. Write a generator that takes another generator and doubles values
> 8. Use sum() with a generator expression
> 9. Write a Countdown generator, use it in a for loop
> 10. What's the memory difference between `[x for x in range(10**7)]` and `(x for x in range(10**7))`?

---

## DAY 4: ADVANCED GENERATORS
**Jun 3, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: yield from, send, generator pipelines
> 00:20-01:10 Hands-on: build processing pipelines
> 01:10-01:30 Drills: advanced generator exercises

### yield from — Delegate to Sub-Generator

```python
# yield from lets you yield all values from another iterable

# Without yield from:
def chain_manual(*iterables):
    for it in iterables:
        for item in it:
            yield item

# With yield from (cleaner):
def chain(*iterables):
    for it in iterables:
        yield from it            # yields each item from it

# Usage:
list(chain([1, 2], [3, 4], [5, 6]))    # [1, 2, 3, 4, 5, 6]

# SRE use: combine log sources
def all_logs():
    yield from read_errors("/var/log/app.log")
    yield from read_errors("/var/log/api.log")
    yield from read_errors("/var/log/worker.log")

for error in all_logs():
    print(error)

# Also useful for recursive generators:
def flatten(nested):
    for item in nested:
        if isinstance(item, list):
            yield from flatten(item)    # recurse!
        else:
            yield item

list(flatten([1, [2, [3, 4]], [5, 6]]))    # [1, 2, 3, 4, 5, 6]
```

### Generator Pipelines — SRE Power Pattern

```python
# Chain generators together like Unix pipes: cmd1 | cmd2 | cmd3
# Each stage processes one item at a time — memory efficient

# Stage 1: Read lines
def read_lines(filepath):
    with open(filepath) as f:
        for line in f:
            yield line.strip()

# Stage 2: Filter
def grep(lines, pattern):
    for line in lines:
        if pattern in line:
            yield line

# Stage 3: Parse
def parse_log(lines):
    for line in lines:
        parts = line.split(None, 3)
        if len(parts) >= 4:
            yield {
                "timestamp": f"{parts[0]} {parts[1]}",
                "level": parts[2],
                "message": parts[3],
            }

# Stage 4: Transform
def add_severity_score(entries):
    scores = {"ERROR": 3, "WARN": 2, "INFO": 1, "DEBUG": 0}
    for entry in entries:
        entry["score"] = scores.get(entry["level"], 0)
        yield entry

# CHAIN THEM (like Unix pipes!):
pipeline = add_severity_score(
    parse_log(
        grep(
            read_lines("/var/log/app.log"),
            "ERROR"
        )
    )
)

# Process one entry at a time through the entire pipeline:
for entry in pipeline:
    print(f"[{entry['score']}] {entry['timestamp']}: {entry['message']}")

# This processes a 100 GB log file with CONSTANT memory usage.
# Each line flows through all stages before the next line enters.
```

### itertools — The Standard Library Power Tools

```python
import itertools

# islice — get a slice of an infinite generator
def infinite_counter():
    n = 0
    while True:
        yield n
        n += 1

first_10 = list(itertools.islice(infinite_counter(), 10))
# [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

# chain — concatenate iterables
all_items = itertools.chain([1, 2], [3, 4], [5, 6])
list(all_items)    # [1, 2, 3, 4, 5, 6]

# groupby — group consecutive items
data = [
    ("ERROR", "timeout"), ("ERROR", "refused"),
    ("INFO", "started"), ("INFO", "ready"),
    ("ERROR", "disk full"),
]
for level, group in itertools.groupby(data, key=lambda x: x[0]):
    entries = list(group)
    print(f"{level}: {len(entries)} entries")

# cycle — repeat forever
servers = itertools.cycle(["web-01", "web-02", "web-03"])
for i, server in zip(range(9), servers):
    print(f"Request {i} → {server}")
# Round-robin load balancing!

# takewhile / dropwhile
readings = [20, 45, 78, 92, 85, 34, 12]
before_spike = list(itertools.takewhile(lambda x: x < 80, readings))
# [20, 45, 78]  — stops at first value >= 80

# product — cartesian product
envs = ["dev", "staging", "prod"]
regions = ["us-east", "us-west"]
for env, region in itertools.product(envs, regions):
    print(f"Deploy to {env}/{region}")
```

### DAY 4 EXERCISES

> **EXERCISE 1: Build a Log Processing Pipeline**
>
> Create these generator stages:
> a) `read_lines(filepath)` → yields stripped lines
> b) `filter_level(lines, level)` → yields matching lines
> c) `extract_hosts(lines)` → yields unique hostnames from log lines
> d) `count_items(items)` → yields (item, count) tuples
> e) Chain them to answer: "Which hosts have the most ERRORs?"

> **EXERCISE 2: yield from Practice**
>
> a) Write `flatten(nested_list)` using yield from recursively
> b) Write `merge_sorted(list1, list2)` that yields merged values in order
> c) Write a generator that delegates to different sub-generators based on input

> **EXERCISE 3: itertools Practice**
>
> a) Use `islice` to get lines 100-200 from a file (without loading all)
> b) Use `cycle` to build a round-robin server assignment
> c) Use `groupby` to group log entries by level
> d) Use `chain` to combine errors from 3 different log files
> e) Use `takewhile` to get all readings before a threshold breach

> **DRILL: 10 Advanced Generator Challenges (3 min each)**
>
> 1. Write yield from to delegate to a sub-generator
> 2. Write a 3-stage generator pipeline
> 3. Use itertools.islice on an infinite generator
> 4. Use itertools.cycle to round-robin through 3 items
> 5. Use itertools.chain to merge two generators
> 6. Flatten `[1, [2, 3], [4, [5, 6]]]` recursively with yield from
> 7. Write a generator that reads 2 files alternately (line from A, line from B)
> 8. Use itertools.groupby on sorted data
> 9. What does `itertools.count(start=10, step=2)` produce?
> 10. Build a pipeline: read file → filter ERRORs → count → top 5

---

## DAY 5: ADVANCED CONTEXT MANAGERS & CAPSTONE
**Jun 4, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: class-based context managers, contextlib, patterns
> 00:20-01:10 Hands-on: capstone combining all week's concepts
> 01:10-01:30 Drills: combined review

### Class-Based Context Manager

```python
# The 'with' statement calls __enter__ and __exit__

class DatabaseConnection:
    """Context manager for database connections."""
    
    def __init__(self, host, port, database):
        self.host = host
        self.port = port
        self.database = database
        self.connection = None
    
    def __enter__(self):
        """Called when entering 'with' block. Returns the resource."""
        print(f"Connecting to {self.host}:{self.port}/{self.database}")
        self.connection = f"conn_{self.host}"   # simulated connection
        return self                              # 'as' variable gets this
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        """Called when leaving 'with' block. Cleanup happens here."""
        print(f"Closing connection to {self.host}")
        self.connection = None
        
        # exc_type, exc_val, exc_tb are exception info (None if no error)
        if exc_type:
            print(f"Error occurred: {exc_type.__name__}: {exc_val}")
        
        # Return True to SUPPRESS the exception
        # Return False (default) to let it propagate
        return False

# Usage:
with DatabaseConnection("db-01", 5432, "myapp") as db:
    print(f"Using connection: {db.connection}")
    # db.connection is automatically closed when leaving this block
    # Even if an exception occurs!

# Output:
# Connecting to db-01:5432/myapp
# Using connection: conn_db-01
# Closing connection to db-01
```

### contextlib Patterns

```python
from contextlib import contextmanager, suppress

# @contextmanager — create context managers from generators
@contextmanager
def managed_connection(host, port):
    """Generator-based context manager."""
    print(f"Connecting to {host}:{port}")
    conn = {"host": host, "port": port, "active": True}
    try:
        yield conn                    # code inside 'with' runs here
    finally:
        print(f"Closing connection to {host}")
        conn["active"] = False

with managed_connection("db-01", 5432) as conn:
    print(f"Connected: {conn}")

# suppress — ignore specific exceptions
from contextlib import suppress

with suppress(FileNotFoundError):
    os.remove("maybe_missing.tmp")
# No error if file doesn't exist — cleaner than try/except


# SRE pattern: temporary directory
@contextmanager
def temp_working_dir(prefix="sre_"):
    """Create a temp directory, clean up when done."""
    import tempfile, shutil
    tmpdir = tempfile.mkdtemp(prefix=prefix)
    original_dir = os.getcwd()
    try:
        os.chdir(tmpdir)
        yield tmpdir
    finally:
        os.chdir(original_dir)
        shutil.rmtree(tmpdir)

with temp_working_dir() as tmpdir:
    # do work in temp dir
    print(f"Working in {tmpdir}")
# temp dir is automatically deleted
```

### CAPSTONE: SRE Monitoring Toolkit

> **Build a toolkit that combines decorators, generators, and context managers.**
>
> **Part 1: Decorators**
> ```python
> @timer                      # measures execution time
> @retry(max_attempts=3)      # retries on failure
> @rate_limit(calls_per_sec=5) # prevents flooding
> def check_server(hostname):
>     """Check if a server is healthy."""
>     pass
> ```
>
> **Part 2: Generator Pipeline**
> ```python
> # Process a log file through multiple stages:
> pipeline = (
>     format_alert(entry)
>     for entry in add_severity(
>         parse_log(
>             filter_level(
>                 read_lines("app.log"),
>                 "ERROR"
>             )
>         )
>     )
>     if entry["score"] >= 2
> )
> ```
>
> **Part 3: Context Manager**
> ```python
> with MonitoringSession("web-tier") as session:
>     for server in ["web-01", "web-02", "web-03"]:
>         result = session.check(server)
>     session.report()
> # Session auto-saves report and cleans up
> ```
>
> **Combine all three:**
> - MonitoringSession context manager sets up the monitoring environment
> - check_server is decorated with @timer, @retry, @rate_limit
> - Results flow through a generator pipeline for processing
> - Final report is written to a file
>
> Write the complete toolkit and test it.

### DAY 5 EXERCISES

> **EXERCISE 1: Class-Based Context Manager**
>
> Build `Timer` context manager:
> a) Records start/end time
> b) Prints elapsed time on exit
> c) Stores elapsed as attribute
> d) Handles exceptions (print error, don't suppress)
> e) Test: `with Timer("sort") as t: sorted(range(100000))`

> **EXERCISE 2: contextlib Practice**
>
> a) Convert your Timer to use @contextmanager decorator
> b) Write a `@contextmanager` that temporarily changes a global config value
> c) Use `suppress` to safely delete files that may not exist
> d) Write a context manager that redirects stdout to a file

> **EXERCISE 3: Full Integration**
>
> Combine everything:
> a) A @timer decorator (Day 2)
> b) A generator pipeline that processes data (Day 3-4)
> c) A context manager that manages output file (Day 5)
> d) Wire them together into a working SRE script

> **DRILL: 10 Final Challenges (3 min each)**
>
> 1. Write a context manager class with __enter__ and __exit__
> 2. Write a @contextmanager generator version
> 3. What arguments does __exit__ receive?
> 4. Return True from __exit__ — what happens to exceptions?
> 5. Use `suppress(KeyError)` to safely access a dict
> 6. Write a decorator that's also a context manager
> 7. Chain a decorator + generator + context manager in one script
> 8. What's the advantage of @contextmanager vs class-based?
> 9. Write a context manager that logs entry and exit
> 10. Explain decorators, generators, and context managers in one sentence each

---

## WEEK 16 SUMMARY

> **FILE 11 COMPLETE — PYTHON ADVANCED I**
>
> ✓ Closures: functions that remember their enclosing scope
> ✓ Decorators: @syntax, functools.wraps, args, stacking, real-world patterns
> ✓ Iterators: __iter__, __next__, iteration protocol
> ✓ Generators: yield, lazy evaluation, memory efficiency
> ✓ Generator pipelines: chain stages like Unix pipes
> ✓ itertools: islice, chain, cycle, groupby, takewhile
> ✓ Context managers: class-based, @contextmanager, suppress
> ✓ 40+ exercises including capstone SRE toolkit
>
> **INTERVIEW PREP:**
> - "What is a decorator?" → A function that wraps another to add behavior.
>   Uses closures. @syntax is sugar for func = decorator(func).
> - "What is a generator?" → A function with yield that produces values
>   lazily, one at a time. Memory efficient for large data.
> - "What is a closure?" → A function that captures variables from its
>   enclosing scope, even after that scope finishes.
> - "What is a context manager?" → An object with __enter__ and __exit__
>   that ensures cleanup. Used with 'with' statement.
> - "What is functools.lru_cache?" → Built-in memoization decorator.
>   Caches function results based on arguments.
>
> Next: File 12 — Python Advanced II (Week 17, Jun 7-11)
> Topics: OOP Deep Dive, Modules, Packages, Virtual Environments
