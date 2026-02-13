# Week 8: Building RAG Pipelines

## 🎯 Learning Objectives

By the end of this week, you will:
- Understand why documents need chunking and how chunk size affects quality
- Implement multiple chunking strategies (recursive, markdown-aware)
- Build a complete end-to-end RAG pipeline: Load → Chunk → Embed → Store → Retrieve → Generate
- Connect your Week 7 vector database to Claude for answer generation
- Handle the "I don't know" case — when no relevant documents exist
- Build a working SRE Q&A system over your runbook collection

---

## 📖 Theory (20 min)

### The Full RAG Pipeline

Last week you built the retrieval half. This week you complete the pipeline:

```
INDEXING (done once):
  Raw Documents → Load → Chunk → Embed → Store in Vector DB

QUERYING (every question):
  User Question → Embed → Search Vector DB → Top-K chunks
       ↓
  Augment Claude's prompt with retrieved chunks
       ↓
  Claude generates answer using YOUR data
```

### Why Chunking Matters — The Most Critical RAG Decision

You can't embed a 50-page runbook as one vector. Here's why:

**Problem 1: Embedding dilution.** An embedding of a whole document captures the AVERAGE meaning. If your document talks about Redis, Kubernetes, AND databases, the embedding is a blur of all three topics. A query about Redis would only partially match.

**Problem 2: Context window waste.** If you retrieve a whole document but only one paragraph is relevant, you're stuffing Claude's prompt with irrelevant text that costs tokens and can confuse the model.

**Solution:** Break documents into **chunks** — smaller pieces that each contain one focused idea.

### Chunking Strategies

**1. Fixed-Size Chunking**
Split every N characters, regardless of content boundaries.
```
"This is a paragraph about Redis failures.\n\nThis is about database | issues."
                                                                      ^ cut here
```
- Pro: Simple, predictable chunk sizes
- Con: Cuts mid-sentence, mid-paragraph, mid-idea

**2. Recursive Character Splitting** ✅ (most commonly used)
Try to split on paragraph breaks first, then sentences, then words.
```
Tries: "\n\n" first → then "\n" → then ". " → then " " → then by character
```
- Pro: Respects natural text boundaries
- Con: Chunk sizes vary

**3. Markdown Header Splitting** ✅ (best for technical docs)
Split on `#`, `##`, `###` headers. Each section becomes a chunk.
```
## Step 1: Check Pods        → chunk 1
## Step 2: Check Database     → chunk 2  
## Step 3: Resolution         → chunk 3
```
- Pro: Preserves document structure perfectly
- Con: Only works for markdown/structured documents

### Chunk Size — Finding the Sweet Spot

| Chunk Size | Retrieval Precision | Context Quality | Token Cost |
|-----------|-------------------|----------------|------------|
| 200 chars | Very high (specific) | Low (missing context) | Low |
| 500 chars | High | Good balance | Medium |
| 1000 chars | Medium | Good context | Higher |
| 2000 chars | Low (too vague) | Full context | High |

**Sweet spot for SRE runbooks: 500-1000 characters.**

### Chunk Overlap — Preventing Context Loss

When you split text, information at the boundary gets separated:

```
Chunk 1: "...increase memory limits to 2Gi."
Chunk 2: "If the issue persists after increasing memory..."
```

A question about "what to do if increasing memory doesn't work" needs BOTH chunks. **Overlap** solves this by sharing text between consecutive chunks:

```
Chunk 1: "...increase memory limits to 2Gi. If the issue persists"
Chunk 2: "If the issue persists after increasing memory, check for..."
                                        ← overlap zone →
```

**Good overlap: 10-20% of chunk_size.** So for 500-char chunks, use 50-100 char overlap.

### The Generation Step — Prompting Claude with Retrieved Context

The key to good RAG generation is a well-structured prompt:

```
System: "Answer using ONLY the provided context. If the context doesn't 
         contain the answer, say 'I don't have a runbook for that.'"

User: "<context>
       [retrieved chunks go here]
       </context>
       
       <question>
       [user's question]
       </question>
       
       Answer the question using only the context above."
```

**Critical rules for the generation prompt:**
1. Tell Claude to use ONLY the provided context (prevents hallucination)
2. Tell Claude to admit when it doesn't know (prevents making stuff up)
3. Ask Claude to cite which document/section it used
4. Use XML tags to clearly separate context from question

---

## 🔨 Hands-On (50 min)

### Exercise 1: Chunking Strategies Deep Dive (15 min)

**Goal:** See how different chunking strategies break up the same document, and understand the tradeoffs.

Create `week8/chunking_strategies.py`:

```python
"""
Week 8, Exercise 1: Chunking Strategy Comparison

See how the same document gets split by different strategies.
This is the most important decision in your RAG pipeline.
"""

# ============================================================
# Our test document — a real SRE runbook
# ============================================================
RUNBOOK = """# Redis Cluster Recovery Runbook

## Overview
This runbook covers recovery procedures for Redis cluster failures affecting the payment-gateway service. Redis is used for rate limiting (redis-ratelimit cluster) and session caching (redis-sessions cluster).

## Prerequisites
- kubectl access to the payments namespace
- redis-cli installed on your workstation
- Access to Grafana dashboard: grafana.internal/d/redis-cluster
- PagerDuty escalation policy: payments-oncall

## Step 1: Assess the Situation
First, determine which Redis cluster is affected and the scope of impact.

Check cluster health:
```
redis-cli -h redis-ratelimit.payments.svc cluster info
redis-cli -h redis-sessions.payments.svc cluster info
```

Expected output for healthy cluster: `cluster_state:ok`
If you see `cluster_state:fail`, proceed to Step 2.

Check which pods are affected:
```
kubectl get pods -n payments -l app=redis-cluster
```

Look for pods in CrashLoopBackOff or NotReady state.

## Step 2: Identify the Failure Mode
Redis cluster failures typically fall into these categories:

### Memory Exhaustion (OOM)
Symptoms: Pod OOMKilled, used_memory near maxmemory
```
redis-cli -h <host> info memory | grep used_memory_human
redis-cli -h <host> CONFIG GET maxmemory
```

### Split Brain
Symptoms: Multiple masters for the same slot range
```
redis-cli -h <host> cluster nodes | grep master
```
If you see multiple masters for overlapping slot ranges, this is a split brain.

### Network Partition
Symptoms: Some nodes can't reach others, cluster_known_nodes < expected
```
redis-cli -h <host> cluster nodes | grep -c connected
```

## Step 3: Recovery Procedures

### For Memory Exhaustion:
1. Increase maxmemory: `redis-cli CONFIG SET maxmemory 4gb`
2. Check eviction policy: `redis-cli CONFIG GET maxmemory-policy`
   Should be `allkeys-lru` for cache use cases
3. If data loss acceptable: `redis-cli FLUSHDB` on affected database
4. Long-term: update StatefulSet to increase memory limits

### For Split Brain:
1. Identify the correct master (most recent data)
2. Force failover: `redis-cli -h <correct-master> CLUSTER FAILOVER TAKEOVER`
3. Remove stale master: `redis-cli CLUSTER FORGET <stale-node-id>`
4. Verify: `redis-cli cluster info` should show cluster_state:ok

### For Network Partition:
1. Check Kubernetes network policies: `kubectl get networkpolicy -n payments`
2. Check CNI plugin health: `kubectl logs -n kube-system -l app=calico-node`
3. Restart affected Redis pods: `kubectl delete pod <pod-name> -n payments`
4. Wait for cluster to reform: watch `redis-cli cluster info`

## Step 4: Verification
After any recovery action:
1. Confirm cluster state: `redis-cli cluster info` → cluster_state:ok
2. Check all nodes: `redis-cli cluster nodes` → all nodes connected
3. Monitor Grafana dashboard for 30 minutes
4. Verify application metrics: error rate and latency returning to normal
5. Check payment-gateway logs for Redis connection errors clearing

## Step 5: Post-Recovery
1. Update incident ticket with actions taken
2. If data was lost (FLUSHDB), application will rebuild cache from database
   Monitor: cache hit rate should recover to >90% within 1 hour
3. Schedule postmortem if recovery took >15 minutes
4. Review and update this runbook if the failure mode was new

## Escalation
- If cluster won't recover after 15 minutes: page secondary on-call
- If split brain can't be resolved: page database team lead
- If payment error rate >5% for >10 minutes: page engineering manager
"""


# ============================================================
# STRATEGY 1: Recursive Character Splitting
# ============================================================
print("=" * 70)
print("STRATEGY 1: Recursive Character Splitting")
print("=" * 70)

def recursive_split(text, chunk_size=500, overlap=80):
    """Simple recursive character splitter."""
    separators = ["\n\n", "\n", ". ", " "]
    chunks = []
    
    def _split(text, sep_idx=0):
        if len(text) <= chunk_size:
            chunks.append(text.strip())
            return
        
        if sep_idx >= len(separators):
            # No more separators — hard split
            chunks.append(text[:chunk_size].strip())
            _split(text[chunk_size - overlap:], 0)
            return
        
        sep = separators[sep_idx]
        parts = text.split(sep)
        
        current = ""
        for part in parts:
            test = current + sep + part if current else part
            if len(test) > chunk_size and current:
                chunks.append(current.strip())
                # Start next chunk with overlap
                overlap_text = current[-(overlap):] if len(current) > overlap else current
                current = overlap_text + sep + part
            else:
                current = test
        
        if current.strip():
            chunks.append(current.strip())
    
    _split(text)
    return chunks

recursive_chunks = recursive_split(RUNBOOK, chunk_size=600, overlap=80)
print(f"\nChunk size=600, overlap=80 → {len(recursive_chunks)} chunks\n")
for i, chunk in enumerate(recursive_chunks):
    first_line = chunk.split('\n')[0][:60]
    print(f"  Chunk {i+1:2d} ({len(chunk):4d} chars): {first_line}...")


# ============================================================
# STRATEGY 2: Markdown Header Splitting
# ============================================================
print(f"\n{'=' * 70}")
print("STRATEGY 2: Markdown Header Splitting")
print("=" * 70)

def markdown_split(text):
    """Split on markdown headers, keeping header as metadata."""
    chunks = []
    current_headers = {}
    current_content = []
    
    for line in text.split('\n'):
        if line.startswith('### '):
            if current_content:
                chunks.append({
                    "headers": dict(current_headers),
                    "content": '\n'.join(current_content).strip()
                })
            current_headers['h3'] = line[4:].strip()
            current_content = [line]
        elif line.startswith('## '):
            if current_content:
                chunks.append({
                    "headers": dict(current_headers),
                    "content": '\n'.join(current_content).strip()
                })
            current_headers = {'h2': line[3:].strip()}
            current_headers.pop('h3', None)
            current_content = [line]
        elif line.startswith('# '):
            if current_content:
                chunks.append({
                    "headers": dict(current_headers),
                    "content": '\n'.join(current_content).strip()
                })
            current_headers = {'h1': line[2:].strip()}
            current_content = [line]
        else:
            current_content.append(line)
    
    if current_content:
        chunks.append({
            "headers": dict(current_headers),
            "content": '\n'.join(current_content).strip()
        })
    
    return chunks

md_chunks = markdown_split(RUNBOOK)
print(f"\nMarkdown splitting → {len(md_chunks)} chunks\n")
for i, chunk in enumerate(md_chunks):
    headers = " > ".join(chunk['headers'].values())
    print(f"  Chunk {i+1:2d} ({len(chunk['content']):4d} chars): [{headers}]")


# ============================================================
# COMPARISON
# ============================================================
print(f"\n{'=' * 70}")
print("COMPARISON")
print("=" * 70)
print(f"""
  Recursive: {len(recursive_chunks)} chunks, average {sum(len(c) for c in recursive_chunks)//len(recursive_chunks)} chars
  Markdown:  {len(md_chunks)} chunks, average {sum(len(c['content']) for c in md_chunks)//len(md_chunks)} chars

  Recursive pros: Consistent chunk sizes, works on any text
  Recursive cons: May split mid-section, losing context

  Markdown pros: Preserves document structure perfectly
  Markdown cons: Variable chunk sizes, only works on structured docs

  For SRE runbooks (which are markdown): Use MARKDOWN splitting ✅
  For raw logs or unstructured text: Use RECURSIVE splitting ✅
""")
```

---

### Exercise 2: Complete RAG Pipeline — Question Answering (25 min)

**Goal:** Build the full pipeline from documents to answers. This is the core deliverable of Phase 3.

Create `week8/rag_pipeline.py`:

```python
"""
Week 8, Exercise 2: Complete RAG Pipeline

The full chain:
  Documents → Chunk → Embed → Store → Retrieve → Augment Prompt → Claude → Answer

This is your SRE Q&A system — ask any question, get answers from YOUR runbooks.
"""
import chromadb
from sentence_transformers import SentenceTransformer
import anthropic
import json


class SREKnowledgeBase:
    """
    A complete RAG-powered SRE knowledge base.
    
    Load your runbooks once, then ask unlimited questions.
    Claude answers using ONLY the content of your runbooks.
    """
    
    def __init__(self, embedding_model: str = 'all-MiniLM-L6-v2'):
        self.embedder = SentenceTransformer(embedding_model)
        self.chroma = chromadb.Client()
        self.collection = self.chroma.create_collection("sre_kb")
        self.claude = anthropic.Anthropic()
        self.doc_count = 0
        self.chunk_count = 0
    
    # ============================================================
    # INDEXING: Load → Chunk → Embed → Store
    # ============================================================
    def add_runbook(self, title: str, content: str, team: str = "sre",
                    chunk_size: int = 600, chunk_overlap: int = 80):
        """
        Add a runbook to the knowledge base.
        Automatically chunks, embeds, and stores it.
        """
        # Chunk the document
        chunks = self._chunk_text(content, chunk_size, chunk_overlap)
        
        # Embed and store each chunk
        for i, chunk in enumerate(chunks):
            chunk_id = f"doc{self.doc_count:03d}-chunk{i:03d}"
            
            # Create a searchable text that includes the title for context
            search_text = f"Runbook: {title}\n\n{chunk}"
            embedding = self.embedder.encode(search_text).tolist()
            
            self.collection.add(
                ids=[chunk_id],
                documents=[chunk],
                metadatas=[{
                    "title": title,
                    "team": team,
                    "chunk_index": i,
                    "total_chunks": len(chunks),
                    "doc_id": self.doc_count
                }],
                embeddings=[embedding]
            )
            self.chunk_count += 1
        
        self.doc_count += 1
        print(f"  📄 Added: \"{title}\" → {len(chunks)} chunks")
    
    def _chunk_text(self, text: str, chunk_size: int, overlap: int) -> list:
        """Recursive character splitting with overlap."""
        if len(text) <= chunk_size:
            return [text.strip()] if text.strip() else []
        
        chunks = []
        start = 0
        
        while start < len(text):
            end = start + chunk_size
            
            if end >= len(text):
                chunk = text[start:].strip()
                if chunk:
                    chunks.append(chunk)
                break
            
            # Try to break at a paragraph boundary
            break_point = text.rfind('\n\n', start, end)
            if break_point == -1 or break_point <= start:
                # Try sentence boundary
                break_point = text.rfind('. ', start, end)
            if break_point == -1 or break_point <= start:
                # Try line boundary
                break_point = text.rfind('\n', start, end)
            if break_point == -1 or break_point <= start:
                # Hard break
                break_point = end
            else:
                break_point += 1  # Include the separator
            
            chunk = text[start:break_point].strip()
            if chunk:
                chunks.append(chunk)
            
            start = break_point - overlap  # Overlap with previous chunk
            if start <= (break_point - chunk_size):  # Prevent infinite loop
                start = break_point
        
        return chunks
    
    # ============================================================
    # QUERYING: Search → Retrieve → Augment → Generate
    # ============================================================
    def ask(self, question: str, n_results: int = 3, team_filter: str = None,
            show_sources: bool = True) -> str:
        """
        Ask a question and get an answer based on your runbooks.
        
        Args:
            question: Natural language question
            n_results: How many chunks to retrieve
            team_filter: Only search docs from this team
            show_sources: Print which sources were used
        
        Returns:
            Claude's answer based on retrieved context
        """
        # STEP 1: Retrieve relevant chunks
        query_embedding = self.embedder.encode(question).tolist()
        
        where_filter = {"team": team_filter} if team_filter else None
        
        results = self.collection.query(
            query_embeddings=[query_embedding],
            n_results=n_results,
            where=where_filter,
            include=["documents", "metadatas", "distances"]
        )
        
        if not results['documents'][0]:
            return "I don't have any runbooks that match your question."
        
        # STEP 2: Build context from retrieved chunks
        context_parts = []
        if show_sources:
            print(f"  📖 Retrieved {len(results['documents'][0])} chunks:")
        
        for i in range(len(results['documents'][0])):
            doc = results['documents'][0][i]
            meta = results['metadatas'][0][i]
            dist = results['distances'][0][i]
            
            context_parts.append(
                f"[Source: {meta['title']} | Chunk {meta['chunk_index']+1}/{meta['total_chunks']}]\n{doc}"
            )
            
            if show_sources:
                relevance = "🟢" if dist < 1.0 else "🟡" if dist < 1.5 else "🔴"
                print(f"     {relevance} {meta['title']} (chunk {meta['chunk_index']+1}, dist={dist:.3f})")
        
        context = "\n\n---\n\n".join(context_parts)
        
        # STEP 3: Generate answer using Claude
        response = self.claude.messages.create(
            model="claude-sonnet-4-5-20250514",
            max_tokens=1500,
            temperature=0,
            system="""You are an SRE assistant answering questions from the team's runbook knowledge base.

RULES:
1. Answer ONLY using information from the provided runbook context
2. If the context doesn't contain the answer, say: "I don't have a runbook covering this. Consider creating one."
3. Include specific commands from the runbooks — copy them exactly
4. Cite which runbook you're using: [Source: Runbook Title]
5. If multiple runbooks are relevant, synthesize information from all of them
6. Be concise but thorough — an on-call engineer at 3 AM needs clear, actionable steps""",
            messages=[{
                "role": "user",
                "content": f"""<runbook_context>
{context}
</runbook_context>

<question>
{question}
</question>

Answer the question using the runbook context above. Include specific commands and cite your sources."""
            }]
        )
        
        return response.content[0].text
    
    def get_stats(self) -> dict:
        """Get knowledge base statistics."""
        return {
            "documents": self.doc_count,
            "chunks": self.chunk_count,
            "collection_size": self.collection.count()
        }


# ============================================================
# LOAD RUNBOOKS INTO THE KNOWLEDGE BASE
# ============================================================

print("=" * 70)
print("BUILDING SRE KNOWLEDGE BASE")
print("=" * 70)

kb = SREKnowledgeBase()

# Runbook 1: Pod CrashLoopBackOff
kb.add_runbook(
    title="Pod CrashLoopBackOff Recovery",
    team="platform",
    content="""When a pod is in CrashLoopBackOff status, follow these steps to diagnose and resolve.

Check pod logs for the previous crashed container:
kubectl logs <pod-name> --previous -n <namespace>

Check pod events for scheduling or resource issues:
kubectl describe pod <pod-name> -n <namespace>

Common causes and fixes:

OOMKilled (Out of Memory):
The container exceeded its memory limit and was killed by the kernel.
Check: kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
Fix: Increase memory limit in the deployment spec.
kubectl edit deployment <deployment-name> -n <namespace>
Change resources.limits.memory to a higher value (e.g., 512Mi to 1Gi).
Verify: kubectl rollout status deployment/<deployment-name>

Image Pull Error:
The container image could not be downloaded from the registry.
Check: kubectl get events -n <namespace> | grep -i pull
Fix: Verify image tag exists, check registry credentials.
kubectl get secret -n <namespace> | grep registry
kubectl create secret docker-registry regcred --docker-server=<registry> --docker-username=<user> --docker-password=<pass>

Application Crash (non-OOM):
The application itself is crashing on startup.
Check logs for stack traces: kubectl logs <pod-name> --previous
Common: missing config, database unavailable, port conflict.
Fix: Check ConfigMaps and Secrets mounted by the pod.
kubectl get configmap -n <namespace>
kubectl describe pod <pod-name> | grep -A5 "Mounts"

Liveness Probe Failure:
The pod starts but fails health checks.
Check probe config: kubectl get deployment <name> -o yaml | grep -A10 livenessProbe
Common issues: wrong port, path, or too-short timeout.
Fix: Increase initialDelaySeconds and timeoutSeconds.

Quick recovery (temporary): kubectl rollout restart deployment/<name>
Rollback to previous version: kubectl rollout undo deployment/<name>"""
)

# Runbook 2: Redis Failure
kb.add_runbook(
    title="Redis Cache Failure Recovery",
    team="data",
    content="""When Redis cache is failing or degraded, it affects rate limiting and session management.

Step 1: Check connectivity
redis-cli -h redis-primary.payments.svc -p 6379 ping
Expected: PONG. If no response, Redis is unreachable.

Step 2: Check memory usage
redis-cli -h <host> info memory
Key metrics:
- used_memory_human: current usage
- maxmemory_human: configured limit
- mem_fragmentation_ratio: should be 1.0-1.5

If memory is full:
redis-cli CONFIG GET maxmemory-policy
Should be 'allkeys-lru' for cache use cases.
Increase maxmemory: redis-cli CONFIG SET maxmemory 4gb

Step 3: Check connections
redis-cli -h <host> info clients
Watch: connected_clients (normal: 50-200), blocked_clients (should be 0)
If too many connections: check for connection leaks in application code.

Step 4: Check cluster health (if Redis Cluster)
redis-cli -h <host> cluster info
Look for: cluster_state:ok (healthy) or cluster_state:fail (problem)
redis-cli cluster nodes — check for disconnected nodes

Step 5: Application impact
Check payment-gateway logs for Redis errors:
kubectl logs -l app=payment-gateway -n payments --tail=100 | grep -i redis
Check if rate limiting is bypassed (security risk!).

Step 6: Emergency recovery
If data loss is acceptable: redis-cli FLUSHALL
Cache will rebuild from database. Monitor cache hit rate — should recover to >90% within 1 hour.

If cluster is broken: delete and recreate the Redis StatefulSet.
kubectl delete statefulset redis-cluster -n payments
kubectl apply -f redis-cluster.yaml
Wait for all pods to rejoin: watch kubectl get pods -l app=redis-cluster"""
)

# Runbook 3: Database Issues
kb.add_runbook(
    title="PostgreSQL Connection and Performance Issues",
    team="data",
    content="""When PostgreSQL database has connection or performance issues.

Connection Pool Exhaustion:
Check current connections:
SELECT count(*), state FROM pg_stat_activity GROUP BY state;

Find idle connections wasting pool space:
SELECT pid, state, query_start, application_name, client_addr
FROM pg_stat_activity 
WHERE state = 'idle' 
ORDER BY query_start ASC;

Kill idle connections older than 10 minutes:
SELECT pg_terminate_backend(pid) 
FROM pg_stat_activity 
WHERE state = 'idle' AND query_start < now() - interval '10 minutes';

Check max connections setting:
SHOW max_connections;  -- default is usually 100-200

If recurring: deploy pgbouncer as a connection pooler.
Recommended pool size per pod: max_connections / number_of_pods * 0.8

Slow Query Investigation:
Find currently running slow queries:
SELECT pid, now() - query_start AS duration, query, state
FROM pg_stat_activity 
WHERE state = 'active' AND now() - query_start > interval '5 seconds'
ORDER BY duration DESC;

Kill a stuck query:
SELECT pg_cancel_backend(<pid>);     -- graceful
SELECT pg_terminate_backend(<pid>);  -- forceful

Check for missing indexes:
SELECT schemaname, tablename, seq_scan, idx_scan
FROM pg_stat_user_tables 
WHERE seq_scan > idx_scan AND seq_scan > 1000
ORDER BY seq_scan DESC;

Check for table bloat:
SELECT tablename, pg_size_pretty(pg_total_relation_size(tablename::regclass)) as size
FROM pg_tables WHERE schemaname = 'public' ORDER BY pg_total_relation_size(tablename::regclass) DESC LIMIT 10;

Replication Lag:
Check on replica: SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;
If lag > 30 seconds: check WAL sender and network between primary and replica.
If lag > 5 minutes: consider failing over reads to primary temporarily."""
)

# Runbook 4: High Latency
kb.add_runbook(
    title="Service High Latency Investigation",
    team="platform",
    content="""When a service's latency exceeds SLO thresholds, follow this investigation procedure.

Step 1: Scope the problem
Is it ALL endpoints or specific routes?
Check Grafana: grafana.internal/d/service-latency, filter by route
If specific route: likely application-level (bad query, missing cache)
If all routes: likely infrastructure (resource saturation, dependency)

Step 2: Check resource saturation
kubectl top pods -n <namespace> -l app=<service>
kubectl top nodes
If CPU > 80%: scale horizontally (add pods) or vertically (increase CPU limit)
If Memory > 85%: check for memory leaks, increase limits

Step 3: Check upstream dependencies
Database: Check pg_stat_activity for slow queries (see PostgreSQL runbook)
Redis: redis-cli --latency -h <host> (should be < 1ms)
External APIs: Check timeout rates in Grafana

Step 4: Check for recent deployments
kubectl rollout history deployment/<service> -n <namespace>
If deployed in last 4 hours, suspect the deployment.
Compare latency graphs before/after deployment.
Rollback: kubectl rollout undo deployment/<service>

Step 5: Check traffic anomalies
Compare current QPS to baseline in Grafana.
If 2x+ normal: possible DDoS, bot traffic, or upstream retry storm.
Check: Are other services experiencing increased traffic too?

Step 6: JVM/Runtime specific (Java services)
Check GC pauses: kubectl logs <pod> | grep "GC pause"
If Full GC > 1s: increase heap size or investigate memory leaks
Capture thread dump: kubectl exec <pod> -- jstack 1 > thread_dump.txt
Capture heap dump: kubectl exec <pod> -- jmap -dump:live,format=b,file=/tmp/heap.bin 1"""
)

# Runbook 5: DNS
kb.add_runbook(
    title="DNS Resolution Failures",
    team="network",
    content="""When DNS resolution fails inside Kubernetes pods.

Step 1: Test from affected pod
kubectl exec -it <pod> -- nslookup kubernetes.default.svc.cluster.local
kubectl exec -it <pod> -- cat /etc/resolv.conf

Expected resolv.conf:
nameserver 10.96.0.10  (CoreDNS ClusterIP)
search <namespace>.svc.cluster.local svc.cluster.local cluster.local

Step 2: Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

If CoreDNS pods are crashing: check resource limits and configmap.
kubectl describe pod -n kube-system -l k8s-app=kube-dns

Step 3: Check CoreDNS performance
kubectl top pods -n kube-system -l k8s-app=kube-dns
If CPU throttled: increase CPU limits in CoreDNS deployment.

Step 4: ndots optimization
Default ndots=5 causes excessive DNS lookups.
For each query like 'redis-host', Kubernetes tries:
  redis-host.namespace.svc.cluster.local
  redis-host.svc.cluster.local
  redis-host.cluster.local
  redis-host.<search-domain>
  redis-host (final attempt)

Fix: Use FQDN with trailing dot: redis-host.payments.svc.cluster.local.
Or set ndots=2 in pod spec: dnsConfig.options: [{name: ndots, value: "2"}]

Step 5: External DNS issues
Check upstream servers: kubectl get configmap coredns -n kube-system -o yaml
Test external resolution: kubectl exec <pod> -- nslookup google.com
If external fails but internal works: upstream DNS server issue.

Emergency workaround: Add hostAliases to pod spec for critical services."""
)

stats = kb.get_stats()
print(f"\n✅ Knowledge base ready: {stats['documents']} docs, {stats['chunks']} chunks\n")

# ============================================================
# ASK QUESTIONS!
# ============================================================

questions = [
    "My pod keeps restarting with out of memory errors. What should I do?",
    "How do I check if Redis is healthy?",
    "Database connections are maxed out. How do I fix this?",
    "Our payment service is really slow. Where do I start investigating?",
    "DNS lookups are failing in our pods. Help!",
    "How do I check for slow PostgreSQL queries?",
    # Edge case: question not covered by any runbook
    "How do I set up a new Kafka cluster?",
]

for q in questions:
    print(f"\n{'=' * 70}")
    print(f"❓ {q}")
    print(f"{'=' * 70}")
    
    answer = kb.ask(q)
    print(f"\n💡 Answer:\n{answer}")

print(f"\n\n{'=' * 70}")
print("RAG PIPELINE SUMMARY")
print("=" * 70)
print(f"""
What just happened:
1. 5 runbooks → chunked into ~{stats['chunks']} pieces
2. Each chunk embedded (text → 384-dim vector)
3. Stored in ChromaDB vector database
4. For each question:
   a. Question embedded
   b. Top 3 most similar chunks retrieved
   c. Chunks sent to Claude as context
   d. Claude answered using ONLY the runbook data

Key observations:
- Claude includes real kubectl commands from the runbooks
- Claude cites which runbook it's using
- For "Kafka cluster" (not in our knowledge base), Claude says it doesn't know
- This is MUCH more reliable than asking Claude from its general knowledge
""")
```

---

### Exercise 3: Configurable RAG with Quality Controls (10 min)

**Goal:** Add quality controls — relevance threshold, source tracking, and "no answer" detection.

Create `week8/rag_quality.py`:

```python
"""
Week 8, Exercise 3: RAG Quality Controls

Production RAG needs:
- Relevance threshold (don't use irrelevant chunks)
- Confidence scoring
- Source attribution
- Graceful "I don't know" handling
"""
import chromadb
from sentence_transformers import SentenceTransformer
import anthropic
import json


def rag_with_quality_controls(
    question: str,
    collection,
    embedder,
    claude_client,
    n_results: int = 5,
    relevance_threshold: float = 1.5,  # Max distance (lower = stricter)
    min_relevant_chunks: int = 1,       # Need at least this many relevant chunks
):
    """
    RAG query with quality controls.
    
    Returns dict with: answer, sources, confidence, relevant_chunks_found
    """
    # Retrieve
    query_embedding = embedder.encode(question).tolist()
    results = collection.query(
        query_embeddings=[query_embedding],
        n_results=n_results,
        include=["documents", "metadatas", "distances"]
    )
    
    # Filter by relevance threshold
    relevant_chunks = []
    all_sources = set()
    
    for i in range(len(results['documents'][0])):
        distance = results['distances'][0][i]
        if distance <= relevance_threshold:
            relevant_chunks.append({
                "content": results['documents'][0][i],
                "title": results['metadatas'][0][i]['title'],
                "distance": distance
            })
            all_sources.add(results['metadatas'][0][i]['title'])
    
    # Quality gate: enough relevant chunks?
    if len(relevant_chunks) < min_relevant_chunks:
        return {
            "answer": "I don't have a runbook that covers this topic. Consider creating one or asking the team directly.",
            "sources": [],
            "confidence": "low",
            "relevant_chunks_found": 0,
            "total_chunks_searched": n_results
        }
    
    # Build context from relevant chunks only
    context = "\n\n---\n\n".join([
        f"[Source: {c['title']}]\n{c['content']}" for c in relevant_chunks
    ])
    
    # Calculate confidence based on distance scores
    avg_distance = sum(c['distance'] for c in relevant_chunks) / len(relevant_chunks)
    if avg_distance < 0.8:
        confidence = "high"
    elif avg_distance < 1.2:
        confidence = "medium"
    else:
        confidence = "low"
    
    # Generate answer
    response = claude_client.messages.create(
        model="claude-sonnet-4-5-20250514",
        max_tokens=1200,
        temperature=0,
        system="""You are an SRE assistant answering from runbooks.
Use ONLY the provided context. Cite sources with [Source: title].
If the context is only partially relevant, note what's covered and what isn't.
Include specific commands from the runbooks.""",
        messages=[{
            "role": "user",
            "content": f"<context>\n{context}\n</context>\n\n<question>{question}</question>"
        }]
    )
    
    return {
        "answer": response.content[0].text,
        "sources": list(all_sources),
        "confidence": confidence,
        "relevant_chunks_found": len(relevant_chunks),
        "total_chunks_searched": n_results,
        "avg_distance": round(avg_distance, 3)
    }


# ============================================================
# TEST with quality controls
# ============================================================

# (Reuse the knowledge base from Exercise 2 — or rebuild it here)
# For a standalone test:
embedder = SentenceTransformer('all-MiniLM-L6-v2')
chroma = chromadb.Client()
collection = chroma.create_collection("quality_test")
claude = anthropic.Anthropic()

# Add a few chunks for testing
test_docs = [
    ("Pod OOMKilled: increase memory limits. kubectl edit deployment to change resources.limits.memory", "Pod CrashLoopBackOff", "platform"),
    ("Redis connectivity: redis-cli ping. Check memory with redis-cli info memory. Eviction policy should be allkeys-lru", "Redis Recovery", "data"),
    ("Database connections: SELECT count(*) FROM pg_stat_activity. Kill idle: pg_terminate_backend(pid)", "PostgreSQL Issues", "data"),
]

for doc, title, team in test_docs:
    emb = embedder.encode(doc).tolist()
    collection.add(
        ids=[f"test-{title[:10]}"],
        documents=[doc],
        metadatas=[{"title": title, "team": team}],
        embeddings=[emb]
    )

# Test queries
test_queries = [
    "Pod keeps crashing with memory errors",         # Should find with HIGH confidence
    "Redis is not responding",                        # Should find with HIGH confidence
    "How do I deploy a machine learning model?",      # NOT in knowledge base
    "What's the weather in Austin?",                  # NOT in knowledge base
]

for q in test_queries:
    print(f"\n{'=' * 60}")
    print(f"❓ {q}")
    result = rag_with_quality_controls(q, collection, embedder, claude)
    print(f"  Confidence: {result['confidence']}")
    print(f"  Sources: {result['sources']}")
    print(f"  Chunks found: {result['relevant_chunks_found']}/{result['total_chunks_searched']}")
    if result.get('avg_distance'):
        print(f"  Avg distance: {result['avg_distance']}")
    print(f"  Answer: {result['answer'][:200]}...")
```

---

## 📝 Drills (20 min)

### Drill 1: Chunk Size Experiment

Modify Exercise 2 to test different chunk sizes. Load the same runbooks with:
- chunk_size=200, overlap=30
- chunk_size=500, overlap=80
- chunk_size=1000, overlap=150
- chunk_size=2000, overlap=200

For each configuration, ask the same 5 questions and compare:
- Which chunks are retrieved?
- Does the answer quality change?
- How many chunks does Claude need to give a complete answer?

Document the sweet spot for your runbooks.

### Drill 2: Add Your Visa Runbooks

Create 2-3 runbooks relevant to your actual work at Visa:
- VDA transaction timeout troubleshooting
- ClickHouse query optimization
- Payment gateway deployment checklist

Add them to the knowledge base and verify they're retrieved correctly.

### Drill 3: Multi-Document Synthesis

Ask a question that requires information from MULTIPLE runbooks:

"The payment gateway is slow, and I suspect it's either a database issue or a Redis issue. How do I determine which one?"

Does the RAG system retrieve chunks from both the PostgreSQL and Redis runbooks? Does Claude synthesize them into a coherent answer?

### Drill 4: Conversation-Aware RAG

Extend the `SREKnowledgeBase` to support multi-turn conversations:

```python
def ask_followup(self, followup: str, previous_question: str, previous_answer: str) -> str:
    """Handle follow-up questions with context from the previous turn."""
    # Strategy: Combine the previous Q&A with the new question
    # to create a better search query
    combined_query = f"{previous_question} {followup}"
    # Search and generate with both the new context AND the previous answer
```

Test:
- "My pod is crashing" → (answer about OOMKilled)
- Follow-up: "How do I increase the memory?" → Should give specific kubectl command

---

## ✅ Week 8 Checklist

Before moving to Week 9:

- [ ] Can explain why chunking matters and how it affects retrieval
- [ ] Can implement recursive and markdown-aware chunking
- [ ] Have a complete RAG pipeline: Load → Chunk → Embed → Store → Retrieve → Generate
- [ ] Claude answers questions using YOUR runbook data (not training data)
- [ ] RAG handles "I don't know" gracefully (no hallucination)
- [ ] Can add quality controls (relevance threshold, confidence scoring)
- [ ] Understand how chunk_size and chunk_overlap affect answer quality

---

## 🧠 Key Concepts from This Week

### RAG Architecture Diagram

```
┌─────────────────────────────────────────────────┐
│  INDEXING (once)                                 │
│                                                  │
│  Runbooks → Chunk (500-1000 chars) → Embed → DB │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│  QUERYING (per question)                         │
│                                                  │
│  Question → Embed → Search DB → Top K chunks     │
│                         ↓                        │
│  Build prompt: system + context + question       │
│                         ↓                        │
│  Claude → Answer (grounded in YOUR data)         │
└─────────────────────────────────────────────────┘
```

### The RAG Quality Formula

```
RAG Quality = Retrieval Quality × Generation Quality

If retrieval finds the wrong chunks → Claude gives wrong answer (garbage in, garbage out)
If retrieval is perfect but prompt is bad → Claude ignores the context or hallucinates

Both parts must be good. Week 7-8 = retrieval. Week 9-10 = making both excellent.
```

### Production RAG Checklist

```
✅ Chunking strategy appropriate for document type
✅ Chunk size tested and optimized (500-1000 chars for runbooks)
✅ Overlap prevents context loss at boundaries (10-20%)
✅ Relevance threshold filters out noise
✅ "I don't know" handling prevents hallucination
✅ Source attribution for every answer
✅ Confidence scoring for quality monitoring
✅ Retrieval evaluation test suite (Week 7)
```

---

*Next week: Advanced RAG Patterns — re-ranking, query transformation, hybrid search, and building a full evaluation framework.*
