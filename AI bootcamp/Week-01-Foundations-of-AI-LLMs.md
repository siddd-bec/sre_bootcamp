# Week 1: Foundations of AI & Large Language Models

## 🎯 Learning Objectives

By the end of this week, you will:
- Understand what LLMs are and how they work (no math needed)
- Know key terms: tokens, context window, temperature, max_tokens, system prompt
- Have your Python environment set up with Anthropic + OpenAI SDKs
- Make your first API call to Claude and understand the response
- Know how to estimate token usage and API costs

---

## 📖 Theory (20 min)

### What is a Large Language Model?

A Large Language Model (LLM) is a neural network trained on massive amounts of text — books, websites, code, documentation. It learns patterns in language so deeply that it can generate new text, answer questions, write code, analyze data, and reason through problems.

**Simple analogy for SREs:** Imagine someone who has read every runbook, every Stack Overflow answer, every man page, every incident report ever written. They can't perfectly recall specific facts, but they understand patterns incredibly well. Ask them "what causes a 504 error?" and they synthesize knowledge from thousands of sources into a useful answer. That's roughly what an LLM does.

**How it actually works (simplified):**
1. You send text (a "prompt") to the model
2. The model predicts the most likely next words, one at a time
3. It keeps generating until it finishes or hits a limit
4. You get back the generated text

It's sophisticated autocomplete — but trained on so much data that "autocomplete" can write code, debug systems, and analyze incidents.

### Key Concepts You Must Know

#### Tokens

LLMs don't read words — they read **tokens**. A token is roughly ¾ of a word, or about 4 characters.

```
"Hello world"           → 2 tokens
"Kubernetes"            → 3 tokens (Ku-ber-netes)
"kubectl get pods"      → 4 tokens
"CrashLoopBackOff"      → 5 tokens
```

**Why tokens matter:**
- You **pay per token** (both input and output)
- Models have a maximum **context window** measured in tokens
- Longer prompts = more tokens = higher cost

#### Context Window

The maximum number of tokens a model can process in one conversation. Think of it as the model's "short-term memory."

| Model | Context Window | Roughly Equals |
|-------|---------------|----------------|
| Claude Haiku 4.5 | 200K tokens | ~150K words / ~500 pages |
| Claude Sonnet 4.5 | 200K tokens | ~150K words / ~500 pages |
| Claude Opus 4.5 | 200K tokens | ~150K words / ~500 pages |
| GPT-4o | 128K tokens | ~96K words / ~320 pages |

**SRE relevance:** Can you fit an entire log file in the context window? A 10,000-line log file with ~100 chars per line ≈ 250K characters ≈ 62K tokens. Yes, it fits easily in Claude's 200K window!

#### Temperature

Controls randomness in the model's output.

| Temperature | Behavior | Use For |
|-------------|----------|---------|
| 0.0 | Deterministic — nearly same output every time | Log analysis, incident response, code, structured output |
| 0.3 | Slight variation | General SRE tasks |
| 0.7 | More creative | Brainstorming, exploring options |
| 1.0 | Maximum randomness | Creative writing (rarely useful for SRE) |

**Rule of thumb for SRE work: Always use 0.0 to 0.3.** You want consistent, reliable answers when debugging production at 3 AM.

#### max_tokens

The maximum number of tokens the model will generate in its response. This is NOT the context window — it's just the output limit.

- Set it based on how long you expect the answer to be
- Too low: response gets cut off mid-sentence (stop_reason = "max_tokens")
- Too high: you might pay for more than you need
- Typical values: 256 for short answers, 1024 for analyses, 4096 for long documents

#### System Prompt

Instructions that tell the model HOW to behave — its persona, rules, and constraints. This is separate from the user's question and is sent at the start of every conversation.

```
System: "You are a senior SRE at Visa. Always consider PCI compliance.
         Respond with specific commands. Format as numbered steps."

User:    "Our payment gateway is returning 504 errors."
```

The system prompt shapes EVERY response in the conversation.

#### Models — Which One to Use?

| Provider | Model | Best For | Speed | Cost |
|----------|-------|---------|-------|------|
| **Anthropic** | Claude Opus 4.5 | Complex reasoning, long analysis | Slow | $$$ |
| **Anthropic** | Claude Sonnet 4.5 | Best balance of speed + quality | Medium | $$ |
| **Anthropic** | Claude Haiku 4.5 | Fast simple tasks, high volume | Fast | $ |
| OpenAI | GPT-4o | General purpose, multimodal | Medium | $$ |
| OpenAI | o1 / o3 | Deep reasoning, math | Slow | $$$ |
| Meta | Llama 3.x | Open-source, self-hosted | Varies | Free* |
| Google | Gemini 2.x | Multimodal, long context | Medium | $$ |

*Free to download, but you pay for compute to run it.

**For this bootcamp, we'll primarily use Claude Sonnet 4.5** — the best balance of quality, speed, and cost.

#### API vs Chat Interface

| | Chat (claude.ai) | API (code) |
|---|---|---|
| How you use it | Type in browser | Send requests from Python |
| Best for | Manual questions, exploration | Automation, building tools |
| Cost | Subscription ($20/month Pro) | Pay per token |
| Power | Limited to one conversation | Build entire applications |

**As an SRE, the API is your power tool.** You'll automate log analysis, incident triage, runbook lookup — things that would take 10 manual conversations.

---

## 🔨 Hands-On (50 min)

### Exercise 1: Environment Setup (15 min)

**Goal:** Get your Python environment ready for AI development.

#### Step 1: Create workspace

```bash
# Create your AI bootcamp workspace
mkdir -p ~/ai-bootcamp/week1
cd ~/ai-bootcamp

# Create a Python virtual environment (keeps dependencies isolated)
python3 -m venv venv
source venv/bin/activate    # On Mac/Linux
# venv\Scripts\activate     # On Windows

# Your terminal prompt should now show (venv)
```

#### Step 2: Install libraries

```bash
# Core AI libraries
pip install anthropic openai python-dotenv

# We'll add more later, but this is all we need for Week 1
```

#### Step 3: Set up API keys

```bash
# Create a .env file to store your keys securely
cat > .env << 'EOF'
ANTHROPIC_API_KEY=sk-ant-your-key-here
OPENAI_API_KEY=sk-your-key-here
EOF

# CRITICAL: Never commit API keys to git!
echo ".env" >> .gitignore
echo "venv/" >> .gitignore
```

**Where to get API keys:**
- **Anthropic:** Go to https://console.anthropic.com → click "API Keys" → "Create Key"
- **OpenAI:** Go to https://platform.openai.com/api-keys → "Create new secret key"
- Both give you free credits to start with

#### Step 4: Verify everything works

Create `week1/verify_setup.py`:

```python
"""
Verify your AI development environment is ready.
Run: python week1/verify_setup.py
"""
import sys

# Check Python version
print(f"Python version: {sys.version}")
assert sys.version_info >= (3, 9), "Need Python 3.9+"
print("  ✅ Python version OK\n")

# Check libraries
try:
    import anthropic
    print(f"  ✅ anthropic SDK version: {anthropic.__version__}")
except ImportError:
    print("  ❌ Run: pip install anthropic")

try:
    import openai
    print(f"  ✅ openai SDK version: {openai.__version__}")
except ImportError:
    print("  ❌ Run: pip install openai")

try:
    from dotenv import load_dotenv
    print("  ✅ python-dotenv installed")
except ImportError:
    print("  ❌ Run: pip install python-dotenv")

# Check API key is set
import os
from dotenv import load_dotenv
load_dotenv()

key = os.getenv("ANTHROPIC_API_KEY", "")
if key.startswith("sk-ant-"):
    print(f"\n  ✅ Anthropic API key found (starts with {key[:10]}...)")
else:
    print("\n  ⚠️  Anthropic API key not found in .env file")
    print("     Add ANTHROPIC_API_KEY=sk-ant-... to your .env file")

print("\n🎉 Setup complete! Ready for Exercise 2.")
```

Run it:
```bash
cd ~/ai-bootcamp
python week1/verify_setup.py
```

---

### Exercise 2: Your First API Call (20 min)

**Goal:** Send a message to Claude, get a response, and understand every field in the response.

Create `week1/first_api_call.py`:

```python
"""
Week 1, Exercise 2: Your First LLM API Call

You'll learn:
- How to create an Anthropic client
- How to send a message to Claude
- How to read the response object
- What input/output tokens mean
"""
import os
from dotenv import load_dotenv
import anthropic

# Load your API key from the .env file
load_dotenv()

# Create an Anthropic client
# It automatically reads ANTHROPIC_API_KEY from your environment
client = anthropic.Anthropic()

# ============================================================
# PART A: Simple message — no system prompt
# ============================================================
print("=" * 60)
print("PART A: Simple message")
print("=" * 60)

response = client.messages.create(
    model="claude-sonnet-4-5-20250514",   # Which model to use
    max_tokens=256,                        # Max output length
    messages=[
        {
            "role": "user",
            "content": "Explain what a 503 Service Unavailable error means. Keep it to 2 sentences."
        }
    ]
)

# The response object has several useful fields
print(f"\nClaude says:\n{response.content[0].text}")

print(f"\n--- Response Metadata ---")
print(f"  Model:          {response.model}")
print(f"  Input tokens:   {response.usage.input_tokens}")
print(f"  Output tokens:  {response.usage.output_tokens}")
print(f"  Total tokens:   {response.usage.input_tokens + response.usage.output_tokens}")
print(f"  Stop reason:    {response.stop_reason}")
# "end_turn" = Claude finished naturally
# "max_tokens" = Response was cut off (increase max_tokens!)

# ============================================================
# PART B: With a system prompt — Claude behaves differently
# ============================================================
print(f"\n{'=' * 60}")
print("PART B: Same question, but with an SRE system prompt")
print("=" * 60)

response_with_system = client.messages.create(
    model="claude-sonnet-4-5-20250514",
    max_tokens=512,
    system=(
        "You are a senior SRE at Visa working on payment processing systems. "
        "Always mention Kubernetes if relevant. "
        "Give specific diagnostic commands an engineer could run. "
        "Keep responses concise and actionable."
    ),
    messages=[
        {
            "role": "user",
            "content": "Explain what a 503 Service Unavailable error means and what to check first."
        }
    ]
)

print(f"\nClaude (as SRE) says:\n{response_with_system.content[0].text}")
print(f"\n  Input tokens:  {response_with_system.usage.input_tokens}")
print(f"  Output tokens: {response_with_system.usage.output_tokens}")

# ============================================================
# PART C: Cost estimation
# ============================================================
print(f"\n{'=' * 60}")
print("PART C: How much did this cost?")
print("=" * 60)

# Claude Sonnet 4.5 pricing (as of 2025):
#   Input:  $3.00 per million tokens
#   Output: $15.00 per million tokens

for label, r in [("Part A", response), ("Part B", response_with_system)]:
    input_cost = r.usage.input_tokens * 3.00 / 1_000_000
    output_cost = r.usage.output_tokens * 15.00 / 1_000_000
    total_cost = input_cost + output_cost
    print(f"\n  {label}:")
    print(f"    Input:  {r.usage.input_tokens:>5} tokens × $3.00/M  = ${input_cost:.6f}")
    print(f"    Output: {r.usage.output_tokens:>5} tokens × $15.00/M = ${output_cost:.6f}")
    print(f"    Total:  ${total_cost:.6f}")

print(f"\n💡 Key takeaway: Each call costs a fraction of a cent!")
print(f"   But at scale (1000s of calls/day), costs add up. Track them.")
```

Run it:
```bash
python week1/first_api_call.py
```

**What to observe:**
1. Part B gives a more specific, SRE-focused answer because of the system prompt
2. Part B uses more input tokens (the system prompt adds tokens!)
3. The cost per call is tiny — but multiply by thousands of daily calls for production

---

### Exercise 3: Experimenting with Parameters (15 min)

**Goal:** See how temperature, max_tokens, and model choice change the output. This builds your intuition for which settings to use.

Create `week1/parameter_experiments.py`:

```python
"""
Week 1, Exercise 3: How parameters change LLM behavior

You'll learn:
- Temperature: deterministic vs creative output
- max_tokens: what happens when it's too low
- Model comparison: Sonnet vs Haiku (speed and quality)
"""
import anthropic
import time

client = anthropic.Anthropic()

question = "A Kubernetes pod is in CrashLoopBackOff. List the top 3 things to check."

# ============================================================
# EXPERIMENT 1: Temperature — 0.0 vs 1.0
# ============================================================
print("=" * 60)
print("EXPERIMENT 1: Temperature (run the same prompt twice at each temp)")
print("=" * 60)

for temp in [0.0, 1.0]:
    print(f"\n--- Temperature = {temp} ---")
    for run in [1, 2]:
        r = client.messages.create(
            model="claude-sonnet-4-5-20250514",
            max_tokens=200,
            temperature=temp,
            messages=[{"role": "user", "content": question}]
        )
        # Show first 150 chars to compare
        text = r.content[0].text.replace("\n", " ")[:150]
        print(f"  Run {run}: {text}...")

print("\n💡 Notice: temp=0.0 gives nearly identical outputs. temp=1.0 varies each time.")
print("   For SRE work: ALWAYS use temp=0.0 to 0.3")

# ============================================================
# EXPERIMENT 2: max_tokens — too low vs adequate
# ============================================================
print(f"\n{'=' * 60}")
print("EXPERIMENT 2: What happens when max_tokens is too low?")
print("=" * 60)

for max_t in [30, 100, 500]:
    r = client.messages.create(
        model="claude-sonnet-4-5-20250514",
        max_tokens=max_t,
        temperature=0,
        messages=[{"role": "user", "content": question}]
    )
    # Check if response was cut off
    cut_off = "🔴 CUT OFF" if r.stop_reason == "max_tokens" else "🟢 COMPLETE"
    print(f"\n  max_tokens={max_t:>3} | {cut_off} | stop_reason={r.stop_reason}")
    print(f"  Output tokens used: {r.usage.output_tokens}")
    print(f"  Response: {r.content[0].text[:100]}...")

print("\n💡 When stop_reason='max_tokens', the response was truncated!")
print("   Always set max_tokens high enough for your expected output.")

# ============================================================
# EXPERIMENT 3: Model comparison — Sonnet vs Haiku
# ============================================================
print(f"\n{'=' * 60}")
print("EXPERIMENT 3: Sonnet (smart) vs Haiku (fast)")
print("=" * 60)

models = [
    ("claude-sonnet-4-5-20250514", "Sonnet 4.5"),
    ("claude-haiku-4-5-20241022", "Haiku 4.5"),
]

for model_id, model_name in models:
    start = time.time()
    r = client.messages.create(
        model=model_id,
        max_tokens=300,
        temperature=0,
        messages=[{"role": "user", "content": question}]
    )
    elapsed = time.time() - start
    
    print(f"\n  --- {model_name} ({elapsed:.1f}s) ---")
    print(f"  Tokens: {r.usage.input_tokens} in / {r.usage.output_tokens} out")
    
    # Calculate cost (Haiku is much cheaper)
    if "sonnet" in model_id:
        cost = r.usage.input_tokens * 3/1e6 + r.usage.output_tokens * 15/1e6
    else:
        cost = r.usage.input_tokens * 0.80/1e6 + r.usage.output_tokens * 4/1e6
    print(f"  Cost: ${cost:.6f}")
    print(f"  Response:\n{r.content[0].text[:200]}...")

print("\n💡 Haiku is faster and cheaper. Use it for simple tasks (classification, short answers).")
print("   Sonnet gives deeper analysis. Use it for complex tasks (incident RCA, code review).")
print("   Choose based on task complexity, not habit!")
```

Run it:
```bash
python week1/parameter_experiments.py
```

---

## 📝 Drills (20 min)

### Drill 1: Vocabulary Check

Without looking at the notes above, write a one-sentence definition for each term:

1. **Token** — 
2. **Context window** — 
3. **Temperature** — 
4. **max_tokens** — 
5. **System prompt** — 
6. **stop_reason** — 

*Check your answers against the theory section. Understanding these terms is essential for everything that follows.*

### Drill 2: Cost Calculation

Grab a calculator (or do mental math):

1. Claude Sonnet pricing: $3 per million input tokens, $15 per million output tokens
   - You send 2,000 input tokens and receive 800 output tokens
   - What's the cost? → $______

2. You have a log file with 5,000 lines, each ~120 characters long
   - Approximately how many tokens is that? (hint: ~4 chars per token) → ______ tokens
   - Does it fit in Claude's 200K context window? → Yes / No
   - If you send this log + get a 1,000-token analysis back, what's the cost? → $______

3. You want to analyze 500 log files per day using Claude
   - Each file is ~50K tokens in, ~1K tokens out
   - Daily cost = $______
   - Monthly cost (30 days) = $______
   - Is Haiku cheaper? What would the Haiku cost be? (Haiku: $0.80/M input, $4/M output)

### Drill 3: Code Modification Challenges

Starting from `first_api_call.py`:

1. **Change the model to Haiku** — Replace the model string and compare the response quality. Is it noticeably worse for this simple question?

2. **Ask an SRE question** — Change the user message to: "Explain the difference between a liveness probe and a readiness probe in Kubernetes." Compare answers with and without the system prompt.

3. **Add a follow-up question** — Modify the messages array to include a conversation:
   ```python
   messages=[
       {"role": "user", "content": "What is a 503 error?"},
       {"role": "assistant", "content": "A 503 error means..."},  # Paste Claude's actual response here
       {"role": "user", "content": "How would I debug this in Kubernetes?"}
   ]
   ```
   Notice how Claude's second answer builds on the first — that's multi-turn conversation.

### Drill 4: Quick Reference Card

Create a file `week1/cheat_sheet.md` with your own quick reference:

```markdown
# Week 1 Cheat Sheet

## API Call Template
(write the minimal code to make an API call)

## Model Selection Guide
- Use _____ for complex analysis
- Use _____ for fast simple tasks

## Temperature Guide
- Use _____ for SRE tasks
- Use _____ for brainstorming

## Cost Formula
Cost = (input_tokens × $/M) + (output_tokens × $/M)
```

Fill this in from memory, then check against the notes.

---

## ✅ Week 1 Checklist

Before moving to Week 2, confirm you can:

- [ ] Explain what an LLM is in your own words
- [ ] Define: token, context window, temperature, max_tokens, system prompt
- [ ] Run Python code that calls the Claude API
- [ ] Read a response object (content, usage, stop_reason)
- [ ] Calculate the approximate cost of an API call
- [ ] Explain when to use Sonnet vs Haiku
- [ ] Explain why temperature=0 matters for SRE tasks

---

## 🔗 Resources

- [Anthropic API Docs](https://docs.anthropic.com/)
- [Anthropic Python SDK](https://github.com/anthropics/anthropic-sdk-python)
- [Anthropic Pricing](https://www.anthropic.com/pricing)
- [Tokenizer Tool](https://console.anthropic.com/tokenizer) — paste text to see token count

---

*Next week: Prompt Engineering Fundamentals — zero-shot, few-shot, chain-of-thought, and the CRISP framework for writing SRE prompts that actually work.*
