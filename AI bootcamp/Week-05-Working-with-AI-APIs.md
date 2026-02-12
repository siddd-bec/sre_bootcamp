# Week 5: Working with AI APIs

## 🎯 Learning Objectives

By the end of this week, you will:
- Master the Anthropic Messages API — every parameter, every option
- Implement streaming responses for real-time output
- Build a production-ready API client with retry, backoff, and cost tracking
- Implement smart model routing (use Haiku for simple tasks, Sonnet for complex ones)
- Understand rate limits, error handling, and API best practices

---

## 📖 Theory (20 min)

### The Anthropic Messages API — Complete Reference

You've been using `client.messages.create()` since Week 1. Now let's understand every parameter:

```python
response = client.messages.create(
    # REQUIRED
    model="claude-sonnet-4-5-20250514",     # Which model brain to use
    max_tokens=1024,                         # Max output length
    messages=[...],                          # Conversation history
    
    # OPTIONAL — Behavior
    system="You are an SRE...",              # System prompt (persona + rules)
    temperature=0,                           # 0=deterministic, 1=random
    top_p=1.0,                               # Nucleus sampling (usually leave at 1)
    top_k=None,                              # Top-K sampling (usually leave unset)
    stop_sequences=["END", "---"],           # Stop generating at these strings
    
    # OPTIONAL — Tool Use (Week 6!)
    tools=[...],                             # Functions the model can call
    tool_choice={"type": "auto"},            # How model picks tools
    
    # OPTIONAL — Metadata
    metadata={"user_id": "siddharth"},       # Track usage per user
)
```

### The Response Object

```python
response.content          # List of content blocks (text, tool_use)
response.content[0].text  # The actual text response
response.model            # Model that was used
response.stop_reason      # Why generation stopped:
                          #   "end_turn" = finished naturally
                          #   "max_tokens" = hit the limit (truncated!)
                          #   "stop_sequence" = hit a stop string
                          #   "tool_use" = model wants to call a tool
response.usage.input_tokens   # Tokens in your prompt
response.usage.output_tokens  # Tokens in the response
```

### Streaming — Why and When

**Without streaming:** You wait 3-15 seconds seeing nothing, then the full response appears.

**With streaming:** Text appears word-by-word as it's generated. The user sees progress immediately.

**When to stream:**
- Long responses (incident analysis, runbooks) — always stream
- Interactive CLI tools — stream for better UX
- Short classifications — don't bother, too fast anyway
- JSON output — do NOT stream (you need the full JSON to parse it)

### Rate Limits and Errors

The Anthropic API has rate limits. When you hit them:

| Error | Code | What Happened | What to Do |
|-------|------|--------------|------------|
| `RateLimitError` | 429 | Too many requests per minute | Wait and retry (exponential backoff) |
| `APIConnectionError` | — | Network issue | Retry after 1 second |
| `APIStatusError` 500+ | 5xx | Anthropic server issue | Retry after 2 seconds |
| `APIStatusError` 400 | 400 | Bad request (your fault) | Fix the request, don't retry |
| `APIStatusError` 401 | 401 | Invalid API key | Check your key, don't retry |
| `AuthenticationError` | 401 | API key problem | Check .env file |

**Exponential backoff:** Wait 1s, then 2s, then 4s, then 8s... This prevents hammering the API when it's overloaded.

### Model Routing — Use the Right Model for the Job

Not every task needs Sonnet. A smart client routes tasks to the cheapest model that can handle them:

| Task | Model | Why |
|------|-------|-----|
| Log classification (P1/P2/P3/P4) | Haiku | Simple, pattern-matching |
| JSON extraction from logs | Haiku | Structured, template-like |
| Incident root cause analysis | Sonnet | Complex reasoning needed |
| Runbook generation | Sonnet | Long, detailed output |
| Postmortem writing | Sonnet | Nuanced analysis |
| Simple yes/no health check | Haiku | Trivial decision |
| Architecture review | Opus | Deep, multi-factor reasoning |

**Cost comparison (per 1M tokens):**

| Model | Input | Output | Relative Cost |
|-------|-------|--------|--------------|
| Haiku 4.5 | $0.80 | $4.00 | 1x (baseline) |
| Sonnet 4.5 | $3.00 | $15.00 | ~4x |
| Opus 4.5 | $15.00 | $75.00 | ~19x |

Using Haiku where possible can cut your costs by 75%+.

---

## 🔨 Hands-On (50 min)

### Exercise 1: Streaming for Real-Time Output (15 min)

**Goal:** Implement streaming for long responses and measure the time-to-first-token improvement.

Create `week5/streaming.py`:

```python
"""
Week 5, Exercise 1: Streaming API Responses

You'll learn:
- How to use the streaming API
- Time-to-first-token (TTFT) vs total time
- When streaming improves UX significantly
- How to collect the full response from a stream
"""
import anthropic
import time

client = anthropic.Anthropic()

SYSTEM = "You are a senior SRE writing a detailed incident root cause analysis."
PROMPT = """Analyze this production incident and provide a comprehensive root cause analysis.

Incident:
- Service: payment-gateway (Kubernetes, 50 pods)
- Symptom: P99 latency spiked from 200ms to 8s
- Duration: 45 minutes
- Trigger: Database migration ran during peak hours
- Impact: 12% of payment transactions timed out
- Resolution: Killed the migration, added missing index

Cover: executive summary, detailed timeline reconstruction, 
5-whys analysis, contributing factors, action items with owners.
"""

# ============================================================
# APPROACH 1: Non-streaming (wait for full response)
# ============================================================
print("=" * 70)
print("NON-STREAMING: Wait for complete response")
print("=" * 70)

start = time.time()
response = client.messages.create(
    model="claude-sonnet-4-5-20250514",
    max_tokens=2000,
    temperature=0,
    system=SYSTEM,
    messages=[{"role": "user", "content": PROMPT}]
)
total_time_sync = time.time() - start

print(f"⏱  Total wait: {total_time_sync:.1f}s (saw NOTHING until this point)")
print(f"📊 Output tokens: {response.usage.output_tokens}")
print(f"📝 First 100 chars: {response.content[0].text[:100]}...")

# ============================================================
# APPROACH 2: Streaming (see output as it generates)
# ============================================================
print(f"\n{'=' * 70}")
print("STREAMING: See output in real-time")
print("=" * 70)

start = time.time()
first_token_time = None
full_text = []
token_count = 0

with client.messages.stream(
    model="claude-sonnet-4-5-20250514",
    max_tokens=2000,
    temperature=0,
    system=SYSTEM,
    messages=[{"role": "user", "content": PROMPT}]
) as stream:
    for text in stream.text_stream:
        # Record time-to-first-token
        if first_token_time is None:
            first_token_time = time.time() - start
            print(f"⚡ First token appeared after: {first_token_time:.2f}s")
            print(f"📝 Streaming output:\n")
        
        # Print each chunk as it arrives
        print(text, end="", flush=True)
        full_text.append(text)

total_time_stream = time.time() - start

# After stream completes, you can get the final message
final_message = stream.get_final_message()

print(f"\n\n{'=' * 70}")
print("COMPARISON")
print("=" * 70)
print(f"""
  Non-streaming:
    Time to first visible output:  {total_time_sync:.1f}s (entire wait)
    Total time:                    {total_time_sync:.1f}s

  Streaming:
    Time to first visible output:  {first_token_time:.2f}s ← MUCH faster perceived speed!
    Total time:                    {total_time_stream:.1f}s

  Actual speed is similar, but perceived speed is MUCH better with streaming.
  For long responses (incident analysis, runbooks), ALWAYS stream.

  After streaming, you still have the full response:
    Full text length: {len(''.join(full_text))} chars
    Output tokens: {final_message.usage.output_tokens}
""")
```

---

### Exercise 2: Production API Client with Model Routing (25 min)

**Goal:** Build a robust client that handles errors, tracks costs, and routes to the right model.

Create `week5/sre_client.py`:

```python
"""
Week 5, Exercise 2: Production-Ready SRE API Client

Features:
- Automatic retry with exponential backoff
- Smart model routing (Haiku for simple, Sonnet for complex)
- Cost tracking across all calls
- Token usage monitoring
- Streaming support
- Error classification and handling
"""
import anthropic
import time
import json
from typing import Optional, Generator
from dataclasses import dataclass, field
from enum import Enum


class TaskComplexity(Enum):
    """Task complexity determines which model to use."""
    SIMPLE = "simple"       # Classification, yes/no, short extraction → Haiku
    MODERATE = "moderate"   # Standard analysis, summaries → Sonnet
    COMPLEX = "complex"     # Deep RCA, architecture review, long generation → Sonnet
    EXPERT = "expert"       # Multi-factor reasoning, critical decisions → Opus


# Model routing table
MODEL_ROUTING = {
    TaskComplexity.SIMPLE:   "claude-haiku-4-5-20241022",
    TaskComplexity.MODERATE: "claude-sonnet-4-5-20250514",
    TaskComplexity.COMPLEX:  "claude-sonnet-4-5-20250514",
    TaskComplexity.EXPERT:   "claude-sonnet-4-5-20250514",  # Use Opus when available/needed
}

# Pricing per million tokens (as of 2025)
PRICING = {
    "claude-haiku-4-5-20241022":  {"input": 0.80, "output": 4.00},
    "claude-sonnet-4-5-20250514": {"input": 3.00, "output": 15.00},
}


@dataclass
class APICallResult:
    """Result of a single API call with metadata."""
    text: str
    model: str
    input_tokens: int
    output_tokens: int
    cost_usd: float
    latency_seconds: float
    stop_reason: str
    attempts: int


@dataclass
class UsageStats:
    """Cumulative usage statistics."""
    total_calls: int = 0
    total_input_tokens: int = 0
    total_output_tokens: int = 0
    total_cost_usd: float = 0.0
    total_errors: int = 0
    calls_by_model: dict = field(default_factory=dict)


class SREClient:
    """
    Production-ready API client for SRE AI tools.
    
    Usage:
        client = SREClient()
        
        # Simple task → auto-routes to Haiku (cheap, fast)
        result = client.ask("Is this log an error? yes or no: ...", complexity=TaskComplexity.SIMPLE)
        
        # Complex task → auto-routes to Sonnet (smart, thorough)
        result = client.ask("Analyze this incident...", complexity=TaskComplexity.COMPLEX)
        
        # Check costs
        print(client.get_usage_stats())
    """
    
    def __init__(self, default_complexity: TaskComplexity = TaskComplexity.MODERATE):
        self.client = anthropic.Anthropic()
        self.default_complexity = default_complexity
        self.stats = UsageStats()
    
    def _calculate_cost(self, model: str, input_tokens: int, output_tokens: int) -> float:
        """Calculate the cost of an API call."""
        pricing = PRICING.get(model, {"input": 3.0, "output": 15.0})
        return (
            input_tokens * pricing["input"] / 1_000_000 +
            output_tokens * pricing["output"] / 1_000_000
        )
    
    def _update_stats(self, result: APICallResult):
        """Update cumulative usage statistics."""
        self.stats.total_calls += 1
        self.stats.total_input_tokens += result.input_tokens
        self.stats.total_output_tokens += result.output_tokens
        self.stats.total_cost_usd += result.cost_usd
        
        model_name = result.model
        if model_name not in self.stats.calls_by_model:
            self.stats.calls_by_model[model_name] = {"calls": 0, "cost": 0.0}
        self.stats.calls_by_model[model_name]["calls"] += 1
        self.stats.calls_by_model[model_name]["cost"] += result.cost_usd
    
    def ask(
        self,
        question: str,
        system: str = "You are a helpful SRE assistant. Be concise and specific.",
        complexity: Optional[TaskComplexity] = None,
        model_override: Optional[str] = None,
        max_tokens: int = 1024,
        temperature: float = 0,
        max_retries: int = 3,
        json_output: bool = False
    ) -> APICallResult:
        """
        Send a question to Claude with automatic model routing and error handling.
        
        Args:
            question: Your question or prompt
            system: System prompt
            complexity: Task complexity (determines model). None = use default.
            model_override: Force a specific model (bypasses routing)
            max_tokens: Max output tokens
            temperature: Response randomness (0 = deterministic)
            max_retries: Number of retry attempts
            json_output: If True, adds JSON instruction and validates output
        
        Returns:
            APICallResult with response text, cost, and metadata
        """
        # Determine model
        if model_override:
            model = model_override
        else:
            comp = complexity or self.default_complexity
            model = MODEL_ROUTING[comp]
        
        # Add JSON instruction if needed
        if json_output:
            system += "\nYou MUST respond with ONLY valid JSON. No markdown, no explanation."
        
        # Retry loop with exponential backoff
        last_error = None
        for attempt in range(max_retries):
            try:
                start = time.time()
                
                response = self.client.messages.create(
                    model=model,
                    max_tokens=max_tokens,
                    temperature=temperature,
                    system=system,
                    messages=[{"role": "user", "content": question}]
                )
                
                latency = time.time() - start
                
                text = response.content[0].text
                
                # Validate JSON if requested
                if json_output:
                    # Try to parse — if it fails, retry
                    try:
                        json.loads(text)
                    except json.JSONDecodeError:
                        # Try stripping markdown fences
                        import re
                        cleaned = re.sub(r'```(?:json)?\s*\n?', '', text)
                        cleaned = cleaned.replace('```', '').strip()
                        json.loads(cleaned)  # Will raise if still invalid
                        text = cleaned
                
                result = APICallResult(
                    text=text,
                    model=response.model,
                    input_tokens=response.usage.input_tokens,
                    output_tokens=response.usage.output_tokens,
                    cost_usd=self._calculate_cost(
                        response.model,
                        response.usage.input_tokens,
                        response.usage.output_tokens
                    ),
                    latency_seconds=round(latency, 2),
                    stop_reason=response.stop_reason,
                    attempts=attempt + 1
                )
                
                self._update_stats(result)
                return result
            
            except anthropic.RateLimitError:
                wait = min(2 ** attempt, 30)  # Cap at 30 seconds
                print(f"⏳ Rate limited. Waiting {wait}s... (attempt {attempt+1}/{max_retries})")
                time.sleep(wait)
                last_error = "Rate limit exceeded"
            
            except anthropic.APIConnectionError as e:
                wait = min(2 ** attempt, 10)
                print(f"🔌 Connection error. Retrying in {wait}s... (attempt {attempt+1}/{max_retries})")
                time.sleep(wait)
                last_error = f"Connection error: {e}"
            
            except anthropic.APIStatusError as e:
                if e.status_code >= 500:
                    wait = min(2 ** attempt, 15)
                    print(f"🔥 Server error {e.status_code}. Retrying in {wait}s...")
                    time.sleep(wait)
                    last_error = f"Server error: {e.status_code}"
                else:
                    # Client errors (400, 401, 403) — don't retry
                    self.stats.total_errors += 1
                    raise RuntimeError(f"API client error {e.status_code}: {e.message}")
            
            except json.JSONDecodeError:
                print(f"⚠️  Invalid JSON. Retrying... (attempt {attempt+1}/{max_retries})")
                last_error = "Invalid JSON response"
                time.sleep(0.5)
        
        # All retries exhausted
        self.stats.total_errors += 1
        raise RuntimeError(f"All {max_retries} retries failed. Last error: {last_error}")
    
    def ask_streaming(
        self,
        question: str,
        system: str = "You are a helpful SRE assistant.",
        model: str = "claude-sonnet-4-5-20250514",
        max_tokens: int = 2000,
    ) -> Generator[str, None, APICallResult]:
        """
        Stream a response, yielding text chunks as they arrive.
        
        Usage:
            for chunk in client.ask_streaming("Analyze this incident..."):
                print(chunk, end="", flush=True)
        """
        start = time.time()
        
        with self.client.messages.stream(
            model=model,
            max_tokens=max_tokens,
            temperature=0,
            system=system,
            messages=[{"role": "user", "content": question}]
        ) as stream:
            for text in stream.text_stream:
                yield text
            
            # After stream ends, get the final message for stats
            final = stream.get_final_message()
        
        latency = time.time() - start
        
        result = APICallResult(
            text="[streamed]",
            model=final.model,
            input_tokens=final.usage.input_tokens,
            output_tokens=final.usage.output_tokens,
            cost_usd=self._calculate_cost(
                final.model, final.usage.input_tokens, final.usage.output_tokens
            ),
            latency_seconds=round(latency, 2),
            stop_reason=final.stop_reason,
            attempts=1
        )
        self._update_stats(result)
    
    def get_usage_stats(self) -> dict:
        """Get cumulative usage statistics."""
        return {
            "total_calls": self.stats.total_calls,
            "total_input_tokens": self.stats.total_input_tokens,
            "total_output_tokens": self.stats.total_output_tokens,
            "total_cost_usd": round(self.stats.total_cost_usd, 6),
            "total_errors": self.stats.total_errors,
            "calls_by_model": self.stats.calls_by_model,
            "avg_cost_per_call": round(
                self.stats.total_cost_usd / max(self.stats.total_calls, 1), 6
            )
        }
    
    def reset_stats(self):
        """Reset usage statistics."""
        self.stats = UsageStats()


# ============================================================
# TEST: Model routing in action
# ============================================================

if __name__ == "__main__":
    client = SREClient()
    
    # ---- SIMPLE TASK → Routes to Haiku (cheap, fast) ----
    print("=" * 70)
    print("TASK 1: Simple classification → Haiku")
    print("=" * 70)
    
    result = client.ask(
        question='Classify this log: "WARN [DiskMonitor] /data at 87% capacity". Return only: ERROR, WARNING, or INFO',
        complexity=TaskComplexity.SIMPLE
    )
    print(f"  Model:    {result.model}")
    print(f"  Response: {result.text.strip()}")
    print(f"  Cost:     ${result.cost_usd:.6f}")
    print(f"  Latency:  {result.latency_seconds}s")
    
    # ---- MODERATE TASK → Routes to Sonnet ----
    print(f"\n{'=' * 70}")
    print("TASK 2: Log analysis → Sonnet")
    print("=" * 70)
    
    result = client.ask(
        question="""Analyze these logs and explain what's happening:
ERROR [payment-svc] Connection refused to redis-primary:6379
ERROR [payment-svc] Connection refused to redis-primary:6379
WARN  [payment-svc] Rate limiter disabled (Redis unavailable)
ERROR [fraud-svc] Velocity check failed: Redis cache miss""",
        complexity=TaskComplexity.MODERATE
    )
    print(f"  Model:    {result.model}")
    print(f"  Response: {result.text[:200].strip()}...")
    print(f"  Cost:     ${result.cost_usd:.6f}")
    print(f"  Latency:  {result.latency_seconds}s")
    
    # ---- JSON OUTPUT ----
    print(f"\n{'=' * 70}")
    print("TASK 3: JSON output with validation → Haiku")
    print("=" * 70)
    
    result = client.ask(
        question="""Classify this alert as JSON:
{"severity": "P1|P2|P3|P4", "service": "string", "action_needed": true|false}

Alert: "CPU at 94% on payment-gateway-03 for 20 minutes" """,
        complexity=TaskComplexity.SIMPLE,
        json_output=True
    )
    print(f"  Model:    {result.model}")
    parsed = json.loads(result.text)
    print(f"  Parsed:   {json.dumps(parsed, indent=2)}")
    print(f"  Cost:     ${result.cost_usd:.6f}")
    
    # ---- COMPLEX TASK → Sonnet with streaming ----
    print(f"\n{'=' * 70}")
    print("TASK 4: Complex analysis with streaming → Sonnet")
    print("=" * 70)
    
    for chunk in client.ask_streaming(
        question="""Brief 5-point root cause analysis:
Payment gateway 504 errors spiked to 5% after v2.3.1 deployment.
Database CPU at 95%. New reporting query doing full table scan on 500M row table.""",
        system="You are a senior SRE. Be concise — 5 bullet points max."
    ):
        print(chunk, end="", flush=True)
    print()
    
    # ---- USAGE STATS ----
    print(f"\n{'=' * 70}")
    print("CUMULATIVE USAGE STATS")
    print("=" * 70)
    
    stats = client.get_usage_stats()
    print(json.dumps(stats, indent=2))
    
    print(f"""
💡 Key observations:
  - Haiku calls are ~4x cheaper than Sonnet
  - Simple tasks routed to Haiku save significant cost at scale
  - If you make 1000 calls/day:
    All Sonnet: ~${stats['avg_cost_per_call'] * 1000:.2f}/day
    Smart routing: ~60% cheaper (most calls are simple)
  
  - The client handles all retries and errors automatically
  - Cost tracking helps you set budgets and detect anomalies
""")
```

---

### Exercise 3: Batch Processing with Rate Limit Awareness (10 min)

**Goal:** Process multiple items efficiently while respecting API rate limits.

Create `week5/batch_processing.py`:

```python
"""
Week 5, Exercise 3: Batch Processing Multiple Items

SRE scenario: You have 20 log files to analyze.
You need to process them all efficiently without hitting rate limits.
"""
import anthropic
import time
import json
from concurrent.futures import ThreadPoolExecutor, as_completed

client = anthropic.Anthropic()

# Simulated batch of log snippets to analyze
log_batches = [
    {"id": "svc-001", "logs": "ERROR [payment-svc] Connection refused to redis:6379"},
    {"id": "svc-002", "logs": "INFO [auth-svc] Health check passed, latency 12ms"},
    {"id": "svc-003", "logs": "ERROR [payment-svc] Transaction timeout after 30s"},
    {"id": "svc-004", "logs": "WARN [disk-monitor] /data at 91% capacity"},
    {"id": "svc-005", "logs": "ERROR [fraud-svc] Model inference timeout: 5000ms"},
    {"id": "svc-006", "logs": "INFO [deployer] Rollout payment-svc:v2.3.2 complete"},
    {"id": "svc-007", "logs": "ERROR [payment-svc] Database deadlock detected on tx table"},
    {"id": "svc-008", "logs": "WARN [cert-monitor] TLS cert expires in 3 days for api.visa.com"},
    {"id": "svc-009", "logs": "ERROR [auth-svc] JWT validation failed: token expired"},
    {"id": "svc-010", "logs": "INFO [hpa] Scaled payment-svc from 10 to 25 replicas"},
]


def analyze_single(item: dict, retry_count: int = 2) -> dict:
    """Analyze a single log entry with retry logic."""
    for attempt in range(retry_count + 1):
        try:
            response = client.messages.create(
                model="claude-haiku-4-5-20241022",  # Haiku for batch = cheap!
                max_tokens=200,
                temperature=0,
                system="You are a log classifier. Return ONLY valid JSON.",
                messages=[{
                    "role": "user",
                    "content": f"""Classify this log as JSON:
{{"id": "{item['id']}", "severity": "P1|P2|P3|P4", "action": "brief action"}}

Log: {item['logs']}"""
                }]
            )
            
            text = response.content[0].text.strip()
            # Try to parse JSON
            try:
                result = json.loads(text)
            except json.JSONDecodeError:
                # Extract JSON from text
                start = text.find('{')
                end = text.rfind('}') + 1
                result = json.loads(text[start:end])
            
            result["tokens"] = response.usage.input_tokens + response.usage.output_tokens
            return result
        
        except anthropic.RateLimitError:
            time.sleep(2 ** attempt)
        except Exception as e:
            if attempt == retry_count:
                return {"id": item["id"], "severity": "UNKNOWN", "error": str(e)}
    
    return {"id": item["id"], "severity": "UNKNOWN", "error": "All retries failed"}


# ============================================================
# APPROACH 1: Sequential (simple but slow)
# ============================================================
print("=" * 70)
print("SEQUENTIAL PROCESSING (simple, safe)")
print("=" * 70)

start = time.time()
sequential_results = []

for item in log_batches:
    result = analyze_single(item)
    sequential_results.append(result)
    severity = result.get("severity", "?")
    action = result.get("action", result.get("error", "?"))
    print(f"  {result['id']}: [{severity}] {action[:60]}")

seq_time = time.time() - start
print(f"\n  Total time: {seq_time:.1f}s for {len(log_batches)} items")
print(f"  Avg per item: {seq_time/len(log_batches):.1f}s")

# ============================================================
# APPROACH 2: Parallel with thread pool (faster!)
# ============================================================
print(f"\n{'=' * 70}")
print("PARALLEL PROCESSING (faster, need rate limit awareness)")
print("=" * 70)

start = time.time()
parallel_results = []

# Use max_workers=5 to stay under rate limits
# Anthropic allows ~50 requests/minute on most plans
with ThreadPoolExecutor(max_workers=5) as executor:
    futures = {
        executor.submit(analyze_single, item): item["id"]
        for item in log_batches
    }
    
    for future in as_completed(futures):
        item_id = futures[future]
        try:
            result = future.result()
            parallel_results.append(result)
            severity = result.get("severity", "?")
            action = result.get("action", result.get("error", "?"))
            print(f"  {result['id']}: [{severity}] {action[:60]}")
        except Exception as e:
            print(f"  {item_id}: ERROR — {e}")

par_time = time.time() - start
print(f"\n  Total time: {par_time:.1f}s for {len(log_batches)} items")
print(f"  Avg per item: {par_time/len(log_batches):.1f}s")
print(f"  Speedup: {seq_time/par_time:.1f}x faster")

# ============================================================
# SUMMARY
# ============================================================
total_tokens = sum(r.get("tokens", 0) for r in parallel_results)
haiku_cost = total_tokens * 0.80 / 1_000_000  # Rough estimate using input pricing

# Count severities
severity_counts = {}
for r in parallel_results:
    sev = r.get("severity", "UNKNOWN")
    severity_counts[sev] = severity_counts.get(sev, 0) + 1

print(f"\n{'=' * 70}")
print("BATCH SUMMARY")
print("=" * 70)
print(f"  Items processed: {len(parallel_results)}")
print(f"  Total tokens: ~{total_tokens}")
print(f"  Estimated cost: ~${haiku_cost:.4f} (using Haiku!)")
print(f"  Severity breakdown: {json.dumps(severity_counts, indent=4)}")

critical = [r for r in parallel_results if r.get("severity") in ("P1", "P2")]
if critical:
    print(f"\n  🚨 {len(critical)} critical items need attention:")
    for r in critical:
        print(f"     [{r['severity']}] {r['id']}: {r.get('action', 'check immediately')}")
```

---

## 📝 Drills (20 min)

### Drill 1: Add Streaming to Your SRE Templates

Open `week3/sre_prompts.py` and add a streaming version of `incident_response`:

```python
def incident_response_streaming(self, service, symptoms, duration, customer_impact, **kwargs):
    """Streaming version — prints analysis in real-time as it generates."""
    # Use client.messages.stream() instead of client.messages.create()
    # Yield text chunks so the caller can print them
    # After streaming, return the full text + usage stats
```

Test it — you should see the incident analysis appear word-by-word.

### Drill 2: Smart Complexity Detector

Create `week5/drill2_auto_route.py`:

Build a function that automatically detects task complexity so you don't have to specify it manually:

```python
def detect_complexity(prompt: str) -> TaskComplexity:
    """
    Auto-detect task complexity based on the prompt.
    
    Rules (implement these):
    - Contains "classify", "yes/no", "true/false" → SIMPLE
    - Less than 100 chars → SIMPLE  
    - Contains "analyze", "investigate", "root cause" → COMPLEX
    - Contains "design", "architect", "review" → EXPERT
    - Default → MODERATE
    """
    pass  # Implement this!
```

Test it with 10 different prompts and verify it routes correctly.

### Drill 3: Cost Budget Enforcer

Add a cost budget feature to `SREClient`:

```python
class SREClient:
    def __init__(self, daily_budget_usd: float = 5.00):
        self.daily_budget = daily_budget_usd
        # ...
    
    def ask(self, ...):
        # Before making the call, check if we're over budget
        if self.stats.total_cost_usd >= self.daily_budget:
            raise BudgetExceededError(
                f"Daily budget ${self.daily_budget} exceeded. "
                f"Current spend: ${self.stats.total_cost_usd:.4f}"
            )
        # ... rest of the method
```

This prevents runaway costs if you have a bug that makes thousands of API calls.

### Drill 4: CLI Tool Using the Client

Create `week5/drill4_sre_cli.py`:

Build a command-line interface for quick SRE tasks:

```bash
# Usage examples:
python sre_cli.py classify "ERROR [db] connection refused"
python sre_cli.py analyze "ERROR [payment-svc] timeout after 30s to redis"
python sre_cli.py incident --service payment-gw --symptoms "504 errors at 5%"
python sre_cli.py stats  # Show usage stats
```

Use `argparse` and the `SREClient` class. Route `classify` to Haiku, `analyze` to Sonnet, and `incident` to Sonnet with streaming.

---

## ✅ Week 5 Checklist

Before moving to Week 6:

- [ ] Can implement streaming for real-time responses
- [ ] Have a working `SREClient` class with retry and backoff
- [ ] Understand model routing — when to use Haiku vs Sonnet
- [ ] Can track token usage and costs across multiple calls
- [ ] Can process batches of items efficiently (sequential + parallel)
- [ ] Understand rate limits and how to handle them
- [ ] Know the cost difference between models (Haiku ~4x cheaper)

---

## 🧠 Key Concepts from This Week

### The Production API Client Checklist

Every production client should have:

```
✅ Retry with exponential backoff
✅ Rate limit handling (429 → wait and retry)
✅ Server error handling (5xx → retry)
✅ Client error handling (4xx → don't retry, fix the request)
✅ Connection error handling (network issues → retry)
✅ Timeout handling (set reasonable timeouts)
✅ Cost tracking (know what you're spending)
✅ Token counting (know how close to context limits)
✅ Model routing (right model for right task)
✅ Logging (track every call for debugging)
```

### Cost Optimization Cheat Sheet

```
1. Route simple tasks to Haiku           → Save ~75%
2. Set max_tokens appropriately          → Don't pay for unused output capacity
3. Cache identical requests              → Save 100% on repeat calls
4. Use batch processing (Week 6 Batch API) → 50% discount
5. Shorten system prompts where possible → Fewer input tokens
6. Truncate long inputs to what's needed → Fewer input tokens
```

### When to Stream vs Not

```
Stream:
  ✅ Long responses (>500 tokens expected)
  ✅ Interactive CLI tools
  ✅ Real-time dashboards
  ✅ Incident analysis (user needs to see progress)

Don't stream:
  ❌ JSON output (need full response to parse)
  ❌ Short classifications (too fast to matter)
  ❌ Batch processing (no human watching)
  ❌ Automated pipelines (just need the final result)
```

---

*Next week: Tool Use (Function Calling) — the most powerful API feature and the bridge to AI Agents. Claude will call YOUR functions to investigate production issues.*
