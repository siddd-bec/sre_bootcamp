# 🤖 GenAI Hands-On Bootcamp for SREs — Master Syllabus

## From Zero → Production AI Tools → Interview Ready

**Student:** Siddharth (SRE at Visa, beginner in AI/GenAI)
**Format:** 1.5 hours/day → 20 min theory | 50 min hands-on | 20 min drills
**Prerequisite:** Basic Python (you already have this!)
**Duration:** 20 weeks (extended from 14 to do justice to every topic)
**Goal:** Build production AI tools, master agentic AI, ace MAG7 interviews

---

## How Each Week Works

| Block | Time | What You Do |
|-------|------|------------|
| Theory | 20 min | Concepts, diagrams, the "why" behind each topic |
| Hands-On | 50 min | Real code — every exercise runs and produces output |
| Drills | 20 min | Quick challenges to lock in the day's learning |

Every exercise builds on the previous one. By Week 20, you'll have a **portfolio of 5 production-grade AI projects**.

---

## Phase 1: Foundations & Prompt Engineering (Weeks 1–4)

*Goal: Understand LLMs, master prompting, get comfortable with AI APIs*

| Week | Topic | What You Build |
|------|-------|---------------|
| 1 | **Foundations of AI & LLMs** | First API call to Claude, token counter, parameter experiments (temperature, max_tokens) |
| 2 | **Prompt Engineering Fundamentals** | Zero-shot vs few-shot vs chain-of-thought comparisons, CRISP framework practice |
| 3 | **Prompt Engineering for SRE** | Reusable SRE prompt template library (incident response, log analysis, runbook generator, postmortem writer) |
| 4 | **Advanced Prompting & Structured Output** | JSON/XML output parser, production guardrails with retry logic, prompt injection defense system |

**Phase 1 Deliverable:** `sre-prompt-toolkit` — A Python library of battle-tested SRE prompt templates with guardrails

**Key Skills Gained:**
- Write effective prompts for any SRE task
- Get structured (JSON) output from LLMs reliably
- Handle errors, validate output, defend against injection
- Understand tokens, costs, and model selection

---

## Phase 2: AI APIs & Tool Use (Weeks 5–6)

*Goal: Master the Anthropic SDK, streaming, and function calling*

| Week | Topic | What You Build |
|------|-------|---------------|
| 5 | **Working with AI APIs** | Streaming responses, multi-turn conversations, production-ready client wrapper with retry/backoff |
| 6 | **Tool Use (Function Calling)** | Claude calls YOUR functions — simulated service health checker, deployment checker, log querier. The agentic loop pattern. |

**Phase 2 Deliverable:** `sre-assistant-cli` — A CLI tool where Claude investigates issues by calling your diagnostic functions

**Key Skills Gained:**
- Anthropic Messages API mastery (streaming, system prompts, stop sequences)
- Tool use / function calling — *the foundation of AI Agents*
- Error handling, rate limiting, cost tracking
- Multi-turn conversation management

---

## Phase 3: RAG — Retrieval-Augmented Generation (Weeks 7–10)

*Goal: Build AI systems that answer questions from YOUR documents*

| Week | Topic | What You Build |
|------|-------|---------------|
| 7 | **Embeddings & Vector Search** | Embedding explorer (visualize similarity), ChromaDB setup, first vector search |
| 8 | **Building RAG Pipelines** | Full pipeline: document loading → chunking → embedding → retrieval → generation. LangChain integration. |
| 9 | **Advanced RAG Patterns** | Query transformation, re-ranking with cross-encoders, hybrid search (keyword + semantic), metadata filtering |
| 10 | **RAG Evaluation & Production** | RAG eval framework (precision, recall, faithfulness), handling edge cases, caching, chunk size optimization |

**Phase 3 Deliverable:** `sre-knowledge-base` — RAG system over your runbooks that on-call engineers can query at 3 AM

**Key Skills Gained:**
- Embeddings and vector similarity (conceptual + hands-on)
- ChromaDB, LangChain document loaders, chunking strategies
- Re-ranking, hybrid search, query transformation
- Evaluating and measuring RAG quality
- Production concerns: caching, stale data, no-result handling

---

## Phase 4: MCP — Model Context Protocol (Weeks 11–13)

*Goal: Connect Claude to external tools and data sources using the standard protocol*

| Week | Topic | What You Build |
|------|-------|---------------|
| 11 | **MCP Concepts & Architecture** | Understand hosts/clients/servers, explore existing MCP servers, configure Claude Desktop with filesystem MCP |
| 12 | **Building MCP Servers (Python)** | Build a custom MCP server that exposes SRE tools (pod status, log queries, metrics) to Claude |
| 13 | **Advanced MCP — Resources, Prompts, Composition** | Add resources (runbooks), prompt templates, compose multiple MCP servers, integrate with your RAG pipeline |

**Phase 3 Deliverable:** `sre-mcp-server` — A custom MCP server that gives Claude access to your infrastructure tools

**Key Skills Gained:**
- MCP architecture (hosts, clients, servers, transport)
- MCP primitives: tools, resources, prompts
- Building production MCP servers in Python
- Composing multiple MCP servers
- Claude Desktop integration

---

## Phase 5: Agentic AI & AI Agents (Weeks 14–16)

*Goal: Build AI agents that reason, plan, use tools, and solve multi-step problems*

| Week | Topic | What You Build |
|------|-------|---------------|
| 14 | **Introduction to AI Agents** | ReAct pattern (Reason + Act), single-agent with tool loop, agent vs chatbot comparison |
| 15 | **Building Production Agents** | Multi-tool agent with memory, planning, error recovery. Agent observability and logging. |
| 16 | **Multi-Agent Systems** | Orchestrator + specialist agents (log analyst, metrics checker, runbook finder). Agent-to-agent communication. |

**Phase 5 Deliverable:** `sre-agent` — An AI agent that investigates incidents by autonomously querying logs, checking metrics, reading runbooks, and proposing fixes

**Key Skills Gained:**
- Agent architectures: ReAct, Plan-and-Execute, Tool-Use loops
- Agent memory (short-term conversation, long-term knowledge)
- Multi-agent orchestration
- Agent observability, debugging, and guardrails
- When to use agents vs simpler approaches

---

## Phase 6: Claude Code & AI-Assisted Development (Weeks 17–18)

*Goal: Use Claude Code for real SRE development workflows*

| Week | Topic | What You Build |
|------|-------|---------------|
| 17 | **Claude Code Fundamentals** | Install & configure, CLI workflows, code generation, file editing, bash command execution |
| 18 | **Claude Code for SRE Automation** | Build monitoring scripts, Terraform modules, Kubernetes manifests, CI/CD pipelines — all using Claude Code as your pair programmer |

**Phase 6 Deliverable:** A collection of SRE automation scripts and infrastructure-as-code built with Claude Code

**Key Skills Gained:**
- Claude Code CLI installation and configuration
- Agentic coding workflows (describe → generate → test → iterate)
- Using Claude Code for infrastructure tasks
- MCP server integration with Claude Code
- When AI-assisted coding helps vs when to code manually

---

## Phase 7: AI Platform Engineering & Capstone (Weeks 19–20)

*Goal: Deploy, monitor, and scale AI systems. Build your capstone project.*

| Week | Topic | What You Build |
|------|-------|---------------|
| 19 | **AI Platform Engineering** | Deploy an AI service with FastAPI, add Prometheus metrics, Grafana dashboards, circuit breakers, cost tracking |
| 20 | **Capstone: AI-Powered Incident Triage Agent** | End-to-end system: MCP server + RAG + multi-agent + monitoring. Takes an alert → investigates → proposes fix → drafts comms. |

**Phase 7 Deliverable:** `incident-triage-agent` — Your portfolio capstone project

**Key Skills Gained:**
- Deploying AI services (FastAPI + Docker)
- Monitoring AI systems (latency, tokens, cost, quality)
- Circuit breakers and graceful degradation
- System design for AI platforms
- Portfolio-ready project for interviews

---

## Tools & Technologies You'll Learn

| Category | Tools |
|----------|-------|
| **AI APIs** | Anthropic Claude API, OpenAI API (for comparison) |
| **Python SDKs** | `anthropic`, `openai`, `langchain`, `sentence-transformers` |
| **Vector DBs** | ChromaDB (learning), pgvector (production discussion) |
| **MCP** | MCP Python SDK, Claude Desktop, custom MCP servers |
| **Agent Frameworks** | Build from scratch (understand internals), then LangGraph overview |
| **AI Coding** | Claude Code CLI, agentic coding workflows |
| **Deployment** | FastAPI, Docker, Prometheus, Grafana |
| **Evaluation** | Custom eval frameworks, RAG metrics, agent benchmarks |

---

## 5 Portfolio Projects You'll Complete

| # | Project | Demonstrates |
|---|---------|-------------|
| 1 | `sre-prompt-toolkit` | Prompt engineering, structured output, guardrails |
| 2 | `sre-knowledge-base` | RAG pipeline, embeddings, evaluation |
| 3 | `sre-mcp-server` | MCP protocol, tool building, Claude integration |
| 4 | `sre-agent` | Agentic AI, multi-agent orchestration |
| 5 | `incident-triage-agent` | Full capstone — everything combined |

---

## Weekly File Delivery Plan

Each week will be delivered as a separate `.md` file with complete, runnable code:

```
Week-01-Foundations-of-AI-LLMs.md
Week-02-Prompt-Engineering-Fundamentals.md
Week-03-Prompt-Engineering-for-SRE.md
Week-04-Advanced-Prompting-Structured-Output.md
Week-05-Working-with-AI-APIs.md
Week-06-Tool-Use-Function-Calling.md
Week-07-Embeddings-Vector-Search.md
Week-08-Building-RAG-Pipelines.md
Week-09-Advanced-RAG-Patterns.md
Week-10-RAG-Evaluation-Production.md
Week-11-MCP-Concepts-Architecture.md
Week-12-Building-MCP-Servers.md
Week-13-Advanced-MCP-Composition.md
Week-14-Introduction-to-AI-Agents.md
Week-15-Building-Production-Agents.md
Week-16-Multi-Agent-Systems.md
Week-17-Claude-Code-Fundamentals.md
Week-18-Claude-Code-SRE-Automation.md
Week-19-AI-Platform-Engineering.md
Week-20-Capstone-Incident-Triage-Agent.md
```

---

## Daily Routine (fits into your existing bootcamp)

| Time Block | Activity | Duration |
|-----------|----------|----------|
| Block 1 | SRE Core — Theory | 20 min |
| Block 2 | SRE Core — Hands-on | 50 min |
| Block 3 | SRE Core — Drills | 20 min |
| Block 4 | **🤖 AI Track — Theory** | **20 min** |
| Block 5 | **🤖 AI Track — Hands-on** | **50 min** |
| Block 6 | **🤖 AI Track — Drills** | **20 min** |
| **Total** | | **3 hours** |

*Or do AI Track on alternate days if 3 hours is too much.*

---

## Milestone Checkpoints

| After Week | You Can... |
|-----------|-----------|
| 4 | Write production-quality prompts, get structured JSON from LLMs, build prompt templates |
| 6 | Call Claude's API programmatically, build CLI tools with function calling |
| 10 | Build RAG systems over any document set, evaluate retrieval quality |
| 13 | Build and deploy MCP servers that give Claude access to your tools |
| 16 | Build AI agents that autonomously investigate production issues |
| 18 | Use Claude Code to accelerate your daily SRE development work |
| 20 | Design, deploy, and monitor production AI systems. Portfolio ready. |

---

*Ready? Let's start with Week 1. Ask me for any week's detailed file and I'll deliver it with full runnable code, exercises, and drills.*
