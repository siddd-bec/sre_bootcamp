# PYTHON TRACK — FILE 14 of 48
## PYTHON FOR SRE II
### HTTP/API Requests, Concurrency (threading/asyncio), Prometheus Client, CLI Tools
**Week 19 | Jun 21 - Jun 25, 2025 | 5 Days | 1.5 hrs/day**

---

> **WHAT THIS WEEK COVERS**
>
> File 13 taught you to parse data. This week you learn to:
> - **Talk to APIs** — every monitoring system has one
> - **Do things in parallel** — check 100 servers, don't wait one by one
> - **Export metrics** — Prometheus is the SRE standard
> - **Build CLI tools** — your scripts need proper arguments
>
> These are the skills that separate scripts from TOOLS.

| Day | Topic | Key Skills |
|-----|-------|-----------|
| 1 | HTTP & API Requests | requests library, REST APIs, error handling |
| 2 | Concurrency: Threading | threading, ThreadPoolExecutor, parallel checks |
| 3 | Concurrency: Async | asyncio, aiohttp, async/await basics |
| 4 | Prometheus Client | Gauges, Counters, Histograms, exposing metrics |
| 5 | CLI Tools & Capstone | argparse, click, full SRE tool |

---

## DAY 1: HTTP & API REQUESTS
**Jun 21, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: requests library, REST basics, error handling
> 00:20-01:10 Hands-on: interact with real APIs
> 01:10-01:30 Drills: API exercises

### The requests Library

```python
# Install: pip install requests
import requests

# GET request (fetch data):
response = requests.get("https://httpbin.org/get")
print(response.status_code)       # 200
print(response.json())             # parsed JSON response
print(response.text)               # raw text
print(response.headers)            # response headers
print(response.elapsed)            # time taken

# POST request (send data):
response = requests.post(
    "https://httpbin.org/post",
    json={"hostname": "web-01", "status": "healthy"},    # sends as JSON
)
print(response.status_code)       # 200
print(response.json())

# Other HTTP methods:
requests.put(url, json=data)       # update resource
requests.delete(url)                # delete resource
requests.patch(url, json=data)     # partial update
```

### Request Options

```python
import requests

# Headers:
response = requests.get(
    "https://api.example.com/servers",
    headers={
        "Authorization": "Bearer YOUR_TOKEN",
        "Content-Type": "application/json",
        "Accept": "application/json",
    }
)

# Query parameters:
response = requests.get(
    "https://api.example.com/servers",
    params={"environment": "prod", "status": "critical"},
)
# Sends: https://api.example.com/servers?environment=prod&status=critical

# Timeout (ALWAYS set this in production!):
response = requests.get(
    "https://api.example.com/health",
    timeout=5,                      # 5 seconds max
)

# Timeout with connect + read separately:
response = requests.get(
    "https://api.example.com/health",
    timeout=(3, 10),                # 3s to connect, 10s to read
)
```

### Error Handling for HTTP

```python
import requests
import logging

logger = logging.getLogger(__name__)

def api_request(method, url, **kwargs):
    """Make an API request with proper error handling."""
    kwargs.setdefault("timeout", 10)
    
    try:
        response = requests.request(method, url, **kwargs)
        
        # Raise for 4xx/5xx status codes:
        response.raise_for_status()
        
        return response
        
    except requests.exceptions.ConnectionError:
        logger.error(f"Connection failed: {url}")
        return None
    except requests.exceptions.Timeout:
        logger.error(f"Request timed out: {url}")
        return None
    except requests.exceptions.HTTPError as e:
        logger.error(f"HTTP error {e.response.status_code}: {url}")
        return None
    except requests.exceptions.RequestException as e:
        logger.error(f"Request failed: {e}")
        return None

# Usage:
response = api_request("GET", "https://api.example.com/health")
if response:
    data = response.json()
    print(data)
```

### SRE API Patterns

```python
import requests
import time
import json

# Pattern 1: Health check endpoint
def check_service_health(base_url, timeout=5):
    """Check a service's health endpoint."""
    try:
        response = requests.get(
            f"{base_url}/health",
            timeout=timeout,
        )
        if response.status_code == 200:
            return {"status": "healthy", "code": 200, "latency_ms": response.elapsed.total_seconds() * 1000}
        return {"status": "unhealthy", "code": response.status_code}
    except requests.exceptions.RequestException as e:
        return {"status": "down", "error": str(e)}

# Pattern 2: Retry with backoff
def api_get_with_retry(url, max_retries=3, timeout=10):
    """GET with exponential backoff."""
    for attempt in range(1, max_retries + 1):
        try:
            response = requests.get(url, timeout=timeout)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt == max_retries:
                raise
            delay = 2 ** (attempt - 1)
            logger.warning(f"Attempt {attempt} failed: {e}. Retry in {delay}s")
            time.sleep(delay)

# Pattern 3: Session (reuse connection — faster for many requests)
def check_multiple_endpoints(base_url, endpoints):
    """Check multiple endpoints using a session."""
    results = {}
    with requests.Session() as session:
        session.headers.update({"Accept": "application/json"})
        session.timeout = 5
        
        for endpoint in endpoints:
            url = f"{base_url}{endpoint}"
            try:
                response = session.get(url)
                results[endpoint] = {
                    "status": response.status_code,
                    "latency_ms": round(response.elapsed.total_seconds() * 1000),
                }
            except requests.exceptions.RequestException as e:
                results[endpoint] = {"status": "error", "error": str(e)}
    return results

# Pattern 4: Paginated API
def get_all_pages(url, params=None):
    """Fetch all pages from a paginated API."""
    params = params or {}
    all_items = []
    page = 1
    
    while True:
        params["page"] = page
        response = requests.get(url, params=params, timeout=10)
        response.raise_for_status()
        data = response.json()
        
        items = data.get("results", [])
        if not items:
            break
        
        all_items.extend(items)
        page += 1
        
        if page > data.get("total_pages", 1):
            break
    
    return all_items
```

### DAY 1 EXERCISES

> **EXERCISE 1: API Client Class**
>
> Build a reusable `APIClient`:
> a) __init__ takes base_url, optional auth_token, timeout
> b) Uses requests.Session for connection reuse
> c) get(endpoint, params) method with error handling
> d) post(endpoint, data) method with error handling
> e) Built-in retry logic (3 attempts)
> f) Logs all requests and responses

> **EXERCISE 2: Health Checker**
>
> Write a script that:
> a) Takes a list of URLs from a JSON config file
> b) Checks each /health endpoint
> c) Records status code and latency
> d) Prints a summary table
> e) Saves results to JSON

> **EXERCISE 3: httpbin Practice**
>
> Use https://httpbin.org to practice (it echoes back what you send):
> a) GET /get with query params
> b) POST /post with JSON body
> c) GET /status/500 — handle the 500 error
> d) GET /delay/3 — handle with a 2-second timeout
> e) GET /headers — send custom headers, verify they arrive

> **DRILL: 10 HTTP Challenges (2 min each)**
>
> 1. Make a GET request and print status code
> 2. Parse a JSON response with .json()
> 3. Send query parameters with params={}
> 4. Set a 5-second timeout
> 5. Handle a ConnectionError
> 6. Use raise_for_status() to catch 4xx/5xx
> 7. Make a POST request with JSON body
> 8. Send a custom header
> 9. Measure request latency with response.elapsed
> 10. Create a requests.Session and make 3 requests

---

## DAY 2: CONCURRENCY — THREADING
**Jun 22, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: why concurrency, threading, ThreadPoolExecutor
> 00:20-01:10 Hands-on: parallel health checks
> 01:10-01:30 Drills: threading challenges

### Why Concurrency?

```python
# PROBLEM: Checking 100 servers sequentially takes forever.

import time

def check_server(hostname):
    """Simulate a health check (takes ~1 second)."""
    time.sleep(1)              # network latency simulation
    return {"host": hostname, "status": "healthy"}

# Sequential — 100 servers × 1 second = 100 seconds!
servers = [f"web-{i:02d}" for i in range(100)]
start = time.time()
results = [check_server(s) for s in servers]
print(f"Sequential: {time.time()-start:.1f}s")    # ~100 seconds

# With threading — all 100 run simultaneously = ~1 second!
# (We'll see how next)
```

### ThreadPoolExecutor — The Easy Way

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import time

def check_server(hostname):
    """Simulate a health check."""
    time.sleep(1)
    return {"host": hostname, "status": "healthy"}

servers = [f"web-{i:02d}" for i in range(100)]

# Run all checks in parallel:
start = time.time()
with ThreadPoolExecutor(max_workers=20) as executor:
    # Submit all tasks:
    futures = {executor.submit(check_server, s): s for s in servers}
    
    # Collect results as they complete:
    results = []
    for future in as_completed(futures):
        hostname = futures[future]
        try:
            result = future.result()
            results.append(result)
        except Exception as e:
            print(f"Error checking {hostname}: {e}")

print(f"Parallel: {time.time()-start:.1f}s")     # ~5 seconds (100/20 = 5 batches)
print(f"Checked {len(results)} servers")
```

> **WHY 20 WORKERS, NOT 100?**
>
> More threads ≠ always faster.
> Each thread uses memory and OS resources.
> For I/O-bound work (network, disk): 20-50 workers is a good range.
> For CPU-bound work: threads DON'T help in Python (see GIL below).

### ThreadPoolExecutor.map() — Simpler Alternative

```python
from concurrent.futures import ThreadPoolExecutor

def check_server(hostname):
    time.sleep(1)
    return {"host": hostname, "status": "healthy"}

servers = [f"web-{i:02d}" for i in range(20)]

# map() — simpler when order matters:
with ThreadPoolExecutor(max_workers=10) as executor:
    results = list(executor.map(check_server, servers))

# Results are in the SAME ORDER as input (unlike as_completed)
for r in results:
    print(r)
```

### Thread Safety — Shared State

```python
import threading
from concurrent.futures import ThreadPoolExecutor

# PROBLEM: Multiple threads modifying shared data = bugs!

counter = 0

def unsafe_increment():
    global counter
    for _ in range(100000):
        counter += 1               # NOT thread-safe!

# This gives WRONG results:
threads = [threading.Thread(target=unsafe_increment) for _ in range(5)]
for t in threads: t.start()
for t in threads: t.join()
print(f"Expected: 500000, Got: {counter}")    # probably less!

# FIX: Use a Lock
counter = 0
lock = threading.Lock()

def safe_increment():
    global counter
    for _ in range(100000):
        with lock:                  # only one thread at a time
            counter += 1

# But BETTER: avoid shared state entirely!
# Use ThreadPoolExecutor and collect results instead:
def compute():
    return sum(1 for _ in range(100000))

with ThreadPoolExecutor(max_workers=5) as executor:
    totals = list(executor.map(lambda _: compute(), range(5)))
total = sum(totals)                  # 500000 — correct!
```

### SRE Pattern: Parallel Health Checks

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import requests
import time
import logging

logger = logging.getLogger(__name__)

def check_endpoint(url, timeout=5):
    """Check a single endpoint."""
    start = time.time()
    try:
        response = requests.get(url, timeout=timeout)
        elapsed = (time.time() - start) * 1000
        return {
            "url": url,
            "status": response.status_code,
            "latency_ms": round(elapsed),
            "healthy": response.status_code == 200,
        }
    except requests.exceptions.Timeout:
        return {"url": url, "status": "timeout", "healthy": False}
    except requests.exceptions.ConnectionError:
        return {"url": url, "status": "connection_error", "healthy": False}

def check_all_endpoints(urls, max_workers=10):
    """Check all endpoints in parallel."""
    results = []
    
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        future_to_url = {executor.submit(check_endpoint, url): url for url in urls}
        
        for future in as_completed(future_to_url):
            result = future.result()
            results.append(result)
            
            # Log immediately:
            status = "✓" if result["healthy"] else "✗"
            logger.info(f"{status} {result['url']} [{result['status']}]")
    
    return results

# Usage:
urls = [
    "http://web-01:8080/health",
    "http://web-02:8080/health",
    "http://db-01:5432/health",
    "http://cache-01:6379/health",
]
results = check_all_endpoints(urls)

# Summary:
healthy = sum(1 for r in results if r["healthy"])
print(f"\n{healthy}/{len(results)} endpoints healthy")
```

### The GIL (Global Interpreter Lock)

```
Python's GIL prevents true parallel CPU execution in threads.

WHAT IT MEANS:
- Threads CAN run I/O operations in parallel (network, disk) ✓
- Threads CANNOT run CPU computations in parallel ✗
- For CPU-bound work, use multiprocessing instead

FOR SRE:
- Health checks, API calls, file reads = I/O bound = threads WORK GREAT
- Number crunching, data analysis = CPU bound = use multiprocessing
- In practice, 90% of SRE work is I/O bound, so threads are fine.
```

### DAY 2 EXERCISES

> **EXERCISE 1: Parallel Server Checker**
>
> a) Create a list of 50 fake server URLs
> b) Write a check function with simulated latency (random 0.1-2s)
> c) Run sequentially — measure time
> d) Run with ThreadPoolExecutor(max_workers=10) — measure time
> e) Compare and print the speedup

> **EXERCISE 2: Parallel Log Downloader**
>
> Simulate downloading logs from multiple servers:
> a) 20 servers, each "download" takes 0.5-2s (use time.sleep + random)
> b) Use ThreadPoolExecutor to download all in parallel
> c) Show progress: "Downloaded 5/20... 10/20..."
> d) Collect all results and print summary

> **EXERCISE 3: Thread-Safe Counter**
>
> Build a `SafeCounter` class:
> a) Uses threading.Lock for thread safety
> b) increment(), decrement(), value() methods
> c) Test with 10 threads each incrementing 10000 times
> d) Verify final value is correct (100000)

> **DRILL: 10 Threading Challenges (3 min each)**
>
> 1. Create a ThreadPoolExecutor, submit 5 tasks
> 2. Use as_completed to get results as they finish
> 3. Use executor.map() for ordered results
> 4. Set max_workers=3, submit 10 tasks — observe batching
> 5. Handle an exception in a future with future.result()
> 6. Time sequential vs parallel execution
> 7. What is the GIL? Does it affect I/O-bound work?
> 8. Use threading.Lock to protect shared state
> 9. Create a function that returns results (no shared state)
> 10. When should you use threads vs multiprocessing?

---

## DAY 3: CONCURRENCY — ASYNC
**Jun 23, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: async/await, asyncio, when to use
> 00:20-01:10 Hands-on: async health checks
> 01:10-01:30 Drills: async exercises

### What Is Async?

```python
# Threading: OS manages switching between threads
# Async: YOUR CODE decides when to switch (at await points)

# Analogy:
# Threading = a restaurant with multiple waiters
# Async = ONE waiter who takes orders, submits to kitchen,
#          takes more orders while food cooks, delivers when ready

# Async is SINGLE-THREADED but handles many I/O operations
# by switching between tasks while waiting for responses.
```

### asyncio Basics

```python
import asyncio

# Regular function:
def regular_greet(name):
    return f"Hello, {name}!"

# Async function (coroutine):
async def async_greet(name):
    await asyncio.sleep(1)          # simulates network wait
    return f"Hello, {name}!"

# Run a single coroutine:
result = asyncio.run(async_greet("Siddharth"))
print(result)                        # Hello, Siddharth!

# 'await' = "pause here, let other tasks run while I wait"
# 'async def' = "this function can be paused and resumed"
```

### Running Tasks Concurrently

```python
import asyncio
import time

async def check_server(hostname):
    """Simulate an async health check."""
    await asyncio.sleep(1)          # simulates 1s network call
    return {"host": hostname, "status": "healthy"}

async def main():
    servers = [f"web-{i:02d}" for i in range(20)]
    
    # Sequential — 20 seconds:
    start = time.time()
    for s in servers:
        await check_server(s)
    print(f"Sequential: {time.time()-start:.1f}s")
    
    # Concurrent — 1 second!
    start = time.time()
    tasks = [check_server(s) for s in servers]
    results = await asyncio.gather(*tasks)
    print(f"Concurrent: {time.time()-start:.1f}s")
    print(f"Results: {len(results)}")

asyncio.run(main())
```

### asyncio.gather vs asyncio.as_completed

```python
import asyncio

async def check_server(hostname, delay):
    await asyncio.sleep(delay)
    return {"host": hostname, "latency": delay}

async def main():
    tasks = [
        check_server("web-01", 0.5),
        check_server("web-02", 2.0),
        check_server("web-03", 0.1),
    ]
    
    # gather — wait for ALL, return in order:
    results = await asyncio.gather(*tasks)
    for r in results:
        print(r)    # order: web-01, web-02, web-03 (input order)
    
    # as_completed — process as each finishes:
    tasks = [
        asyncio.create_task(check_server("web-01", 0.5)),
        asyncio.create_task(check_server("web-02", 2.0)),
        asyncio.create_task(check_server("web-03", 0.1)),
    ]
    for coro in asyncio.as_completed(tasks):
        result = await coro
        print(result)    # order: web-03, web-01, web-02 (fastest first)

asyncio.run(main())
```

### aiohttp — Async HTTP Requests

```python
# Install: pip install aiohttp
import aiohttp
import asyncio

async def fetch_url(session, url):
    """Fetch a URL asynchronously."""
    try:
        async with session.get(url, timeout=aiohttp.ClientTimeout(total=5)) as response:
            status = response.status
            data = await response.text()
            return {"url": url, "status": status, "size": len(data)}
    except asyncio.TimeoutError:
        return {"url": url, "status": "timeout"}
    except aiohttp.ClientError as e:
        return {"url": url, "status": "error", "error": str(e)}

async def check_all(urls):
    """Check all URLs concurrently."""
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_url(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
    return results

# Usage:
urls = ["https://httpbin.org/get", "https://httpbin.org/delay/1"]
results = asyncio.run(check_all(urls))
for r in results:
    print(r)
```

### Threading vs Async — When to Use Which

```
THREADING (concurrent.futures):
  ✓ Simpler to understand and debug
  ✓ Works with existing synchronous libraries (requests, etc.)
  ✓ Good for moderate concurrency (10-50 parallel tasks)
  ✗ Each thread uses ~8 MB of memory
  ✗ Thread switching has overhead

ASYNC (asyncio):
  ✓ Very efficient — handles 1000s of connections easily
  ✓ Low memory — single thread, no thread overhead
  ✓ Great for high-concurrency I/O (web servers, monitoring)
  ✗ Need async-compatible libraries (aiohttp instead of requests)
  ✗ Harder to debug ("colored function" problem)
  ✗ Can't use regular blocking code in async functions

FOR SRE:
  - Checking 10-50 servers? → ThreadPoolExecutor (simpler)
  - Checking 500+ endpoints? → asyncio (more scalable)
  - Building a monitoring daemon? → asyncio
  - Quick automation script? → ThreadPoolExecutor
```

### DAY 3 EXERCISES

> **EXERCISE 1: Async Health Checker**
>
> a) Write async functions to check 20 endpoints
> b) Use asyncio.gather to run all concurrently
> c) Measure total time (should be ~= slowest single check)
> d) Print results as they come with as_completed
> e) Compare with the threading version from Day 2

> **EXERCISE 2: Rate-Limited Async**
>
> Write an async function that:
> a) Checks 100 URLs
> b) Limits to 10 concurrent requests (use asyncio.Semaphore)
> c) ```python
>    sem = asyncio.Semaphore(10)
>    async def limited_fetch(url):
>        async with sem:
>            return await fetch_url(session, url)
>    ```
> d) Measure time with different semaphore limits (5, 10, 20)

> **EXERCISE 3: Async Pipeline**
>
> Build an async pipeline:
> a) async fetch_metrics(hostname) → returns metrics dict
> b) async evaluate_health(metrics) → returns health status
> c) async send_alert(hostname, status) → simulates alert
> d) Chain them: for each server, fetch → evaluate → alert if needed
> e) Run all servers concurrently

> **DRILL: 10 Async Challenges (3 min each)**
>
> 1. Write an async function with await asyncio.sleep(1)
> 2. Run it with asyncio.run()
> 3. Use asyncio.gather to run 5 coroutines concurrently
> 4. Use asyncio.create_task to create a task
> 5. Use asyncio.Semaphore to limit concurrency to 3
> 6. Handle a timeout with asyncio.wait_for(coro, timeout=2)
> 7. What's the difference between await and asyncio.gather?
> 8. Can you use requests library inside async def? (no — use aiohttp)
> 9. When is async better than threading?
> 10. What does the event loop do?

---

## DAY 4: PROMETHEUS CLIENT
**Jun 24, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: Prometheus metric types, client library
> 00:20-01:10 Hands-on: instrument an application
> 01:10-01:30 Drills: metrics exercises

### Prometheus Metric Types

```
Prometheus is the standard monitoring system in SRE.
It PULLS metrics from your applications via HTTP.

FOUR METRIC TYPES:

1. COUNTER — only goes UP (or resets to 0)
   Example: total_requests, total_errors, bytes_sent
   "How many X have happened?"

2. GAUGE — goes up AND down (current value)
   Example: current_cpu, memory_usage, active_connections
   "What is the current value of X?"

3. HISTOGRAM — distribution of values (bucketed)
   Example: request_duration, response_size
   "How are values of X distributed?"

4. SUMMARY — similar to histogram but calculates percentiles
   Example: request_duration_p99
   (prefer histograms — they're more flexible)
```

### Using the Prometheus Client Library

```python
# Install: pip install prometheus-client
from prometheus_client import Counter, Gauge, Histogram, Summary
from prometheus_client import start_http_server, generate_latest
import time
import random

# COUNTER — total requests processed:
REQUEST_COUNT = Counter(
    "sre_requests_total",                  # metric name
    "Total number of requests processed",  # help text
    ["method", "endpoint", "status"],      # labels
)

# GAUGE — current CPU usage:
CPU_USAGE = Gauge(
    "sre_cpu_usage_percent",
    "Current CPU usage percentage",
    ["hostname"],
)

# HISTOGRAM — request duration:
REQUEST_DURATION = Histogram(
    "sre_request_duration_seconds",
    "Request processing time in seconds",
    ["endpoint"],
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0],
)

# Use them:
def handle_request(method, endpoint):
    """Simulate handling a request."""
    start = time.time()
    
    # Process...
    time.sleep(random.uniform(0.01, 0.5))
    status = random.choice(["200", "200", "200", "500"])
    
    # Record metrics:
    REQUEST_COUNT.labels(method=method, endpoint=endpoint, status=status).inc()
    REQUEST_DURATION.labels(endpoint=endpoint).observe(time.time() - start)

def update_cpu(hostname, value):
    """Update CPU gauge."""
    CPU_USAGE.labels(hostname=hostname).set(value)
```

### Exposing Metrics via HTTP

```python
from prometheus_client import start_http_server, generate_latest
import time
import random

# Start metrics server on port 8000:
start_http_server(8000)
print("Metrics server on http://localhost:8000/metrics")

# Simulate application work:
while True:
    handle_request("GET", "/api/health")
    handle_request("POST", "/api/data")
    update_cpu("web-01", random.uniform(20, 95))
    time.sleep(1)

# Visit http://localhost:8000/metrics to see:
# # HELP sre_requests_total Total number of requests processed
# # TYPE sre_requests_total counter
# sre_requests_total{method="GET",endpoint="/api/health",status="200"} 42.0
# sre_requests_total{method="GET",endpoint="/api/health",status="500"} 3.0
# ...
```

### Histogram in Detail

```python
from prometheus_client import Histogram
import time

# Histogram tracks the DISTRIBUTION of values.
# Prometheus stores them in buckets.

REQUEST_DURATION = Histogram(
    "http_request_duration_seconds",
    "HTTP request duration",
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0],
)

# Record values:
REQUEST_DURATION.observe(0.045)    # 45ms request
REQUEST_DURATION.observe(0.120)    # 120ms request
REQUEST_DURATION.observe(2.500)    # 2.5s request (slow!)

# Convenient decorator for timing functions:
@REQUEST_DURATION.time()
def process_request():
    time.sleep(random.uniform(0.01, 0.5))

# The histogram generates these metrics:
# http_request_duration_seconds_bucket{le="0.005"} 0
# http_request_duration_seconds_bucket{le="0.01"} 0
# http_request_duration_seconds_bucket{le="0.025"} 0
# http_request_duration_seconds_bucket{le="0.05"} 1      ← 0.045 fits here
# http_request_duration_seconds_bucket{le="0.1"} 1
# http_request_duration_seconds_bucket{le="0.25"} 2      ← 0.120 fits here
# http_request_duration_seconds_bucket{le="+Inf"} 3      ← all values fit
# http_request_duration_seconds_count 3
# http_request_duration_seconds_sum 2.665
```

### SRE Metrics Best Practices

```python
# NAMING: <namespace>_<name>_<unit>
# Good: sre_request_duration_seconds
# Bad:  request_time

# LABELS: Use for dimensions you'll filter/group by
# Good: labels=["method", "endpoint", "status"]
# Bad:  labels=["request_id"]  ← too many unique values (high cardinality)

# THE FOUR GOLDEN SIGNALS (Google SRE book):
# 1. Latency    → Histogram (request_duration_seconds)
# 2. Traffic    → Counter (requests_total)
# 3. Errors     → Counter (errors_total) or label on requests
# 4. Saturation → Gauge (cpu_usage, memory_usage, queue_depth)

from prometheus_client import Counter, Gauge, Histogram

# Golden signals implementation:
LATENCY = Histogram(
    "app_request_duration_seconds",
    "Request latency",
    ["method", "endpoint"],
)

TRAFFIC = Counter(
    "app_requests_total",
    "Total requests",
    ["method", "endpoint"],
)

ERRORS = Counter(
    "app_errors_total",
    "Total errors",
    ["method", "endpoint", "error_type"],
)

SATURATION = Gauge(
    "app_active_connections",
    "Current active connections",
    ["server"],
)
```

### DAY 4 EXERCISES

> **EXERCISE 1: Instrument a Server Monitor**
>
> Create metrics for a server monitoring tool:
> a) Counter: health_checks_total (labels: hostname, result)
> b) Gauge: server_cpu_percent, server_memory_percent (labels: hostname)
> c) Histogram: health_check_duration_seconds
> d) Simulate checking 5 servers in a loop
> e) Start metrics server, visit /metrics in browser

> **EXERCISE 2: Four Golden Signals**
>
> Implement all four golden signals:
> a) Latency histogram for request processing
> b) Traffic counter for total requests
> c) Error counter with error_type label
> d) Saturation gauge for active connections
> e) Simulate traffic and view the metrics output

> **EXERCISE 3: Decorator Metrics**
>
> Build a `@timed` decorator that:
> a) Uses a Histogram to record function duration
> b) Uses a Counter to count function calls
> c) Uses a Counter to count function errors
> d) Labels with function name
> e) Apply to 3 different functions

> **DRILL: 10 Prometheus Challenges (3 min each)**
>
> 1. Create a Counter and increment it
> 2. Create a Gauge and set/inc/dec it
> 3. Create a Histogram with custom buckets
> 4. Add labels to a metric
> 5. Use @histogram.time() to time a function
> 6. Start a metrics HTTP server
> 7. Print generate_latest() to see raw metric output
> 8. What's the difference between Counter and Gauge?
> 9. Why use Histogram over Summary?
> 10. Name the four golden signals

---

## DAY 5: CLI TOOLS & CAPSTONE
**Jun 25, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: argparse, click, building proper CLI tools
> 00:20-01:10 Hands-on: capstone project
> 01:10-01:30 Drills: final review

### argparse — Standard Library CLI

```python
import argparse

def main():
    parser = argparse.ArgumentParser(
        description="SRE Server Health Checker"
    )
    
    # Required argument:
    parser.add_argument(
        "servers",
        nargs="+",                          # one or more
        help="Server hostnames to check"
    )
    
    # Optional arguments:
    parser.add_argument(
        "-t", "--timeout",
        type=int, default=5,
        help="Timeout in seconds (default: 5)"
    )
    parser.add_argument(
        "-w", "--workers",
        type=int, default=10,
        help="Number of parallel workers (default: 10)"
    )
    parser.add_argument(
        "-o", "--output",
        choices=["text", "json", "csv"],
        default="text",
        help="Output format (default: text)"
    )
    parser.add_argument(
        "-v", "--verbose",
        action="store_true",
        help="Enable verbose output"
    )
    parser.add_argument(
        "--config",
        type=str,
        help="Path to config file"
    )
    
    args = parser.parse_args()
    
    # Use the arguments:
    print(f"Checking: {args.servers}")
    print(f"Timeout: {args.timeout}s")
    print(f"Workers: {args.workers}")
    print(f"Output: {args.output}")
    print(f"Verbose: {args.verbose}")

if __name__ == "__main__":
    main()

# Usage:
# python3 checker.py web-01 web-02 web-03 -t 10 -w 20 -o json -v
# python3 checker.py --help
```

### click — Better CLI Library

```python
# Install: pip install click
import click

@click.command()
@click.argument("servers", nargs=-1, required=True)
@click.option("-t", "--timeout", default=5, help="Timeout in seconds")
@click.option("-w", "--workers", default=10, help="Parallel workers")
@click.option("-o", "--output", type=click.Choice(["text", "json", "csv"]), default="text")
@click.option("-v", "--verbose", is_flag=True, help="Verbose output")
@click.option("--config", type=click.Path(exists=True), help="Config file")
def check(servers, timeout, workers, output, verbose, config):
    """Check health of SRE servers."""
    click.echo(f"Checking {len(servers)} servers...")
    for server in servers:
        click.echo(f"  {server}: healthy")

if __name__ == "__main__":
    check()

# click advantages over argparse:
# - Decorator-based (cleaner)
# - click.Path(exists=True) validates file exists
# - click.Choice for valid options
# - Built-in color support: click.secho("ERROR", fg="red")
# - Progress bars: click.progressbar
```

### CAPSTONE: SRE Health Check Tool

> **Build a complete, production-quality CLI tool that combines EVERYTHING from this week.**
>
> **Features:**
> ```
> $ python3 sre-checker.py web-01 web-02 db-01 --timeout 10 --workers 5 --output json
>
> Checking 3 servers with 5 workers (timeout: 10s)
>
> ✓ web-01  [200] 45ms
> ✓ web-02  [200] 32ms
> ✗ db-01   [timeout] -
>
> Results: 2/3 healthy
>
> # Also exposes Prometheus metrics at :8000/metrics
> ```
>
> **Components to build:**
> 1. CLI with argparse or click (Day 5)
> 2. HTTP health checks with requests (Day 1)
> 3. Parallel execution with ThreadPoolExecutor (Day 2)
> 4. Prometheus metrics: check_duration, check_total, check_errors (Day 4)
> 5. Output as text, JSON, or CSV (File 13)
> 6. Proper logging (File 12)
> 7. Config file support (JSON/YAML)
> 8. Retry with backoff on failures
> 9. Error handling throughout
>
> **Project structure:**
> ```
> sre-checker/
> ├── requirements.txt
> ├── config.yaml
> ├── sre_checker/
> │   ├── __init__.py
> │   ├── cli.py            (argparse/click interface)
> │   ├── checker.py         (health check logic)
> │   ├── metrics.py         (Prometheus metrics)
> │   ├── output.py          (text/json/csv formatters)
> │   └── config.py          (config loader)
> └── tests/
>     ├── test_checker.py
>     └── test_output.py
> ```

### DAY 5 EXERCISES

> **EXERCISE 1: Build the CLI**
>
> Using argparse or click:
> a) Accept server hostnames (positional, one or more)
> b) --timeout, --workers, --output flags
> c) --config for config file path
> d) --verbose flag
> e) --help shows usage info
> f) Validate inputs (timeout > 0, workers > 0)

> **EXERCISE 2: Build the Capstone**
>
> Wire everything together:
> a) CLI parses arguments
> b) Config file loaded (with defaults)
> c) Servers checked in parallel
> d) Results formatted as requested (text/json/csv)
> e) Metrics exposed via Prometheus
> f) Logging throughout
> g) Handle ALL errors gracefully

> **EXERCISE 3: Add Tests**
>
> Write tests for:
> a) Config loading (valid, missing, invalid)
> b) Output formatting (text, json, csv)
> c) Health check logic (success, timeout, error)
> d) CLI argument parsing
> e) Run with pytest

> **DRILL: 10 Final Challenges (3 min each)**
>
> 1. Create an argparse parser with 3 options
> 2. Use click to create a CLI command with options
> 3. Combine requests + ThreadPoolExecutor
> 4. Create a Prometheus Counter with labels
> 5. Output results as JSON
> 6. Handle a timeout in requests.get()
> 7. Use logging instead of print
> 8. Read config from YAML
> 9. Create a proper project structure with __init__.py
> 10. Write a pytest test for a function that raises on bad input

---

## WEEK 19 SUMMARY

> **FILE 14 COMPLETE — PYTHON FOR SRE II**
>
> ✓ HTTP/APIs: requests library, sessions, retry, error handling
> ✓ Threading: ThreadPoolExecutor, parallel health checks, GIL
> ✓ Async: asyncio, await, gather, aiohttp, semaphores
> ✓ Prometheus: Counter, Gauge, Histogram, labels, four golden signals
> ✓ CLI tools: argparse, click, proper argument handling
> ✓ Capstone: full SRE health check tool with all components
> ✓ 40+ exercises combining all topics
>
> **INTERVIEW PREP:**
> - "How do you check multiple servers efficiently?" → ThreadPoolExecutor
>   for moderate concurrency, asyncio for 100s+ of endpoints
> - "What are the four golden signals?" → Latency, Traffic, Errors, Saturation
> - "Threading vs async?" → Threading: simpler, works with sync libs.
>   Async: more scalable, needs async libs. Both for I/O-bound work.
> - "What is the GIL?" → Global Interpreter Lock prevents true CPU
>   parallelism in threads. Doesn't affect I/O-bound work.
> - "How do you expose Prometheus metrics?" → prometheus_client library,
>   start_http_server, Counter/Gauge/Histogram with labels
>
> Next: File 15 — Python for SRE III (Week 20, Jun 28 - Jul 2)
> Topics: Automation Scripts, Ansible Integration, ChatOps Basics
