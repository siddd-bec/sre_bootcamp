# Week 7: Embeddings & Vector Search

## 🎯 Learning Objectives

By the end of this week, you will:
- Understand what embeddings are and why they matter (no math required)
- See how text becomes vectors and how similarity search works
- Set up ChromaDB (a local vector database)
- Build a searchable knowledge base from SRE runbooks
- Measure retrieval quality and understand when vector search fails
- Know the difference between keyword search and semantic search

---

## 📖 Theory (20 min)

### The Problem: Claude Doesn't Know YOUR Systems

Claude knows about Kubernetes in general. But it doesn't know:
- YOUR specific runbooks for handling Redis failures
- YOUR team's on-call procedure
- YOUR incident history and what fixed things last time
- YOUR service architecture and connection strings

**RAG (Retrieval-Augmented Generation)** solves this by:
1. Storing YOUR documents in a searchable database
2. Finding the most relevant documents when a question is asked
3. Feeding those documents to Claude along with the question
4. Claude answers using YOUR data, not just its training

This week we focus on steps 1-2: **storing and searching**. Next week we add Claude (step 3-4).

### What Are Embeddings?

An embedding converts text into a **list of numbers** (called a vector) that captures the **meaning** of the text. Texts with similar meaning produce similar vectors.

```
"Kubernetes pod crash"   → [0.12, -0.45, 0.87, 0.23, ...]  (384 numbers)
"Container OOMKilled"    → [0.11, -0.43, 0.85, 0.21, ...]  (very similar!)
"Best pizza in NYC"      → [-0.67, 0.23, -0.12, 0.55, ...]  (very different)
```

**Key insight:** "OOMKilled" and "out of memory" have completely different words but similar meanings. Traditional keyword search would MISS this connection. Embedding-based search finds it.

### How Similarity Works

Two vectors are compared using **cosine similarity** — a score from -1 to 1:
- **1.0** = identical meaning
- **0.7-0.9** = very similar (probably relevant)
- **0.3-0.6** = somewhat related
- **0.0** = completely unrelated

You don't need to understand the math. Just know: higher score = more similar meaning.

### Embedding Models

An embedding model converts text to vectors. Different models produce different quality embeddings:

| Model | Dimensions | Quality | Speed | Cost |
|-------|-----------|---------|-------|------|
| `all-MiniLM-L6-v2` | 384 | Good | Fast | Free (local) |
| `all-mpnet-base-v2` | 768 | Better | Medium | Free (local) |
| OpenAI `text-embedding-3-small` | 1536 | Very good | Fast | $0.02/1M tokens |
| Voyage `voyage-3` | 1024 | Excellent | Fast | $0.06/1M tokens |

**For learning, we use `all-MiniLM-L6-v2`** — it runs locally (no API key needed), is fast, and good enough for our purposes.

### Vector Database — Where Embeddings Live

A vector database is optimized for storing and searching embeddings. When you search, it finds the K vectors most similar to your query.

```
Your documents → embed each one → store vectors in DB
                                        ↓
Your question → embed the question → find K most similar vectors → return those documents
```

| Database | Type | Best For |
|----------|------|---------|
| **ChromaDB** | Local, in-memory or persistent | Learning, prototyping ✅ |
| pgvector | PostgreSQL extension | Teams already using Postgres |
| Pinecone | Cloud managed | Production, auto-scaling |
| Weaviate | Cloud or self-hosted | Production, flexible |
| Qdrant | Cloud or self-hosted | High performance |

### Keyword Search vs Semantic Search

| | Keyword Search (traditional) | Semantic Search (embeddings) |
|---|---|---|
| How it works | Match exact words | Match meaning |
| "OOM" finds "Out of Memory"? | ❌ No | ✅ Yes |
| "pod crash" finds "container CrashLoopBackOff"? | ❌ No | ✅ Yes |
| Speed | Very fast | Fast (with index) |
| Best for | Exact terms, log grep | Natural language questions |

In practice, the best systems use **both** (hybrid search — covered in Week 9).

---

## 🔨 Hands-On (50 min)

### Exercise 1: Visualizing Embeddings & Similarity (15 min)

**Goal:** See how text becomes vectors and understand similarity scores intuitively.

First, install the required libraries:

```bash
pip install sentence-transformers chromadb numpy
```

Create `week7/embeddings_explorer.py`:

```python
"""
Week 7, Exercise 1: Understanding Embeddings

You'll see:
- How text becomes a vector of numbers
- Why similar texts have similar vectors
- Where semantic search succeeds and keyword search fails
"""
import numpy as np
from sentence_transformers import SentenceTransformer

# Load a local embedding model (downloads ~80MB on first run)
print("Loading embedding model...")
model = SentenceTransformer('all-MiniLM-L6-v2')
print("✅ Model loaded\n")

# ============================================================
# PART A: See what an embedding looks like
# ============================================================
print("=" * 70)
print("PART A: What does an embedding look like?")
print("=" * 70)

text = "Kubernetes pod is in CrashLoopBackOff"
embedding = model.encode(text)

print(f"\n  Text: \"{text}\"")
print(f"  Vector dimensions: {len(embedding)}")
print(f"  First 10 values: {embedding[:10].round(4)}")
print(f"  Min value: {embedding.min():.4f}")
print(f"  Max value: {embedding.max():.4f}")
print(f"\n  This text is now represented as {len(embedding)} numbers.")
print(f"  Every text gets mapped to the same {len(embedding)}-dimensional space.")

# ============================================================
# PART B: Similarity between SRE concepts
# ============================================================
print(f"\n{'=' * 70}")
print("PART B: Semantic similarity between SRE texts")
print("=" * 70)

def cosine_similarity(a, b):
    """Calculate similarity between two vectors. Returns 0-1."""
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))

# Pairs of texts to compare
pairs = [
    # Should be HIGH similarity (same concept, different words)
    ("Pod is OOMKilled",
     "Container ran out of memory",
     "Same concept, different words"),
    
    ("Database connection pool exhausted",
     "PostgreSQL max_connections limit reached",
     "Same root cause, different phrasing"),
    
    ("Service returning 504 Gateway Timeout",
     "Backend request timed out after 30 seconds",
     "Same symptom, technical vs descriptive"),
    
    # Should be MODERATE similarity (related but different)
    ("Kubernetes pod crash",
     "Docker container restart",
     "Related technologies"),
    
    ("High CPU usage on web server",
     "Database queries running slow",
     "Both performance issues, different components"),
    
    # Should be LOW similarity (unrelated)
    ("Pod CrashLoopBackOff due to OOMKilled",
     "How to make sourdough bread",
     "Completely unrelated"),
    
    ("TLS certificate expiring in 3 days",
     "Best restaurants in Austin Texas",
     "Completely unrelated"),
]

print(f"\n  {'Score':>6}  {'Bar':<25}  Description")
print(f"  {'─'*6}  {'─'*25}  {'─'*40}")

for text_a, text_b, description in pairs:
    emb_a = model.encode(text_a)
    emb_b = model.encode(text_b)
    score = cosine_similarity(emb_a, emb_b)
    
    bar_length = int(score * 25)
    bar = "█" * bar_length + "░" * (25 - bar_length)
    
    if score >= 0.7:
        emoji = "🟢"
    elif score >= 0.4:
        emoji = "🟡"
    else:
        emoji = "🔴"
    
    print(f"  {emoji}{score:>5.3f}  {bar}  {description}")
    print(f"         A: \"{text_a[:50]}\"")
    print(f"         B: \"{text_b[:50]}\"")
    print()

# ============================================================
# PART C: Semantic search beats keyword search
# ============================================================
print(f"{'=' * 70}")
print("PART C: Where semantic search wins over keyword search")
print("=" * 70)

documents = [
    "When a pod is OOMKilled, increase the memory limit in the deployment spec",
    "To fix CrashLoopBackOff, check pod logs with kubectl logs --previous",
    "Database connection pool exhaustion: check pg_stat_activity for idle connections",
    "Redis cache miss rate above 50%: review eviction policy and maxmemory setting",
    "TLS certificate renewal: use cert-manager for automatic rotation",
    "High latency: check upstream dependencies and recent deployments",
]

queries = [
    "container keeps running out of memory",    # Should match OOMKilled doc
    "too many database connections",             # Should match connection pool doc
    "SSL cert is about to expire",              # Should match TLS cert doc
]

doc_embeddings = model.encode(documents)

for query in queries:
    query_embedding = model.encode(query)
    
    # Calculate similarity with each document
    scores = [cosine_similarity(query_embedding, doc_emb) for doc_emb in doc_embeddings]
    
    # Find the best match
    best_idx = np.argmax(scores)
    best_score = scores[best_idx]
    
    print(f"\n  Query: \"{query}\"")
    print(f"  Best match (score={best_score:.3f}):")
    print(f"    → \"{documents[best_idx]}\"")
    
    # Would keyword search have found this?
    query_words = set(query.lower().split())
    doc_words = set(documents[best_idx].lower().split())
    common_words = query_words & doc_words - {"the", "a", "is", "to", "in", "and", "for"}
    
    if common_words:
        print(f"  Keyword overlap: {common_words}")
    else:
        print(f"  ⭐ Keyword overlap: NONE — only semantic search found this!")

print(f"""
\n💡 Key takeaways:
  1. Embeddings capture MEANING, not just words
  2. "OOMKilled" and "out of memory" are similar in embedding space
  3. "SSL cert" matches "TLS certificate" because they mean the same thing
  4. Traditional grep/keyword search would miss these connections
  5. This is why RAG uses embeddings — your on-call engineer can ask
     natural language questions and find the right runbook
""")
```

---

### Exercise 2: Build a Searchable Runbook Knowledge Base (25 min)

**Goal:** Store SRE runbooks in ChromaDB and search them with natural language.

Create `week7/runbook_search.py`:

```python
"""
Week 7, Exercise 2: Searchable Runbook Knowledge Base

Build a vector database of SRE runbooks that you can query
with natural language questions.

This is the RETRIEVAL part of RAG (Retrieval-Augmented Generation).
Next week, we add the GENERATION part (Claude answers using retrieved docs).
"""
import chromadb
from sentence_transformers import SentenceTransformer
import json

# ============================================================
# STEP 1: Create the runbook knowledge base
# ============================================================

runbooks = [
    {
        "id": "rb-001",
        "title": "Pod CrashLoopBackOff Recovery",
        "team": "platform",
        "content": """When a pod is in CrashLoopBackOff:
1. Check pod logs: kubectl logs <pod-name> --previous
2. Check events: kubectl describe pod <pod-name>
3. Common causes:
   - OOMKilled: increase memory limits in deployment spec
     kubectl edit deployment <name>, change resources.limits.memory
   - Image pull errors: check registry credentials and image tag
     kubectl get events | grep -i pull
   - Application crash: check logs for stack traces
   - Liveness probe failure: review probe config and timeouts
4. Quick restart: kubectl rollout restart deployment/<name>
5. If persists: check if recent deployment introduced the issue
   kubectl rollout history deployment/<name>
   kubectl rollout undo deployment/<name>"""
    },
    {
        "id": "rb-002",
        "title": "Database Connection Pool Exhaustion",
        "team": "data",
        "content": """When database connections are exhausted:
1. Check current connections:
   SELECT count(*), state FROM pg_stat_activity GROUP BY state;
2. Find idle connections:
   SELECT pid, state, query_start, application_name 
   FROM pg_stat_activity WHERE state = 'idle'
   ORDER BY query_start ASC;
3. Check pool settings: max_pool_size should be < max_connections / number_of_pods
4. Kill long-idle connections:
   SELECT pg_terminate_backend(pid) FROM pg_stat_activity 
   WHERE state = 'idle' AND query_start < now() - interval '10 minutes';
5. If recurring: add pgbouncer as connection pooler
6. Alert config: trigger at 80% of max_connections"""
    },
    {
        "id": "rb-003",
        "title": "High Latency Investigation",
        "team": "platform",
        "content": """When service latency exceeds SLO:
1. Determine scope: all endpoints or specific routes?
   Check Grafana dashboard filtered by route/endpoint
2. Check upstream dependencies:
   - Database: EXPLAIN ANALYZE on slow queries
   - Redis: redis-cli --latency to measure cache response time
   - External APIs: check timeout and error rates
3. Check resource saturation:
   kubectl top pods -n <namespace>
   kubectl top nodes
4. Check for recent deployments:
   kubectl rollout history deployment/<name>
5. Check for traffic spikes:
   Compare current QPS to baseline in Grafana
6. If deployment-related: kubectl rollout undo deployment/<name>
7. If resource-related: kubectl scale deployment/<name> --replicas=<n>"""
    },
    {
        "id": "rb-004",
        "title": "TLS Certificate Expiry Response",
        "team": "security",
        "content": """When TLS certificates are expiring or expired:
1. Check expiry date:
   echo | openssl s_client -connect host:443 2>/dev/null | openssl x509 -noout -dates
2. Check cert-manager status:
   kubectl get certificates -A
   kubectl describe certificate <name> -n <namespace>
3. Check CertificateRequest and challenges:
   kubectl get certificaterequest -A
   kubectl get challenges -A
4. If cert-manager is stuck:
   kubectl delete certificate <name> (it will recreate)
   Check issuer: kubectl describe clusterissuer letsencrypt-prod
5. Manual cert update (emergency):
   kubectl create secret tls <name> --cert=cert.pem --key=key.pem -n <ns> --dry-run=client -o yaml | kubectl apply -f -
6. Prevention: cert-manager with auto-renewal, alert at 30 days before expiry"""
    },
    {
        "id": "rb-005",
        "title": "Redis Cache Failure Recovery",
        "team": "data",
        "content": """When Redis cache is failing or degraded:
1. Check connectivity: redis-cli -h <host> -p 6379 ping (expect PONG)
2. Check memory: redis-cli info memory
   Watch: used_memory vs maxmemory, mem_fragmentation_ratio
3. If OOM: check eviction policy
   redis-cli CONFIG GET maxmemory-policy (should be allkeys-lru for cache)
4. Check client connections: redis-cli info clients
   Watch: connected_clients, blocked_clients
5. If cluster: redis-cli cluster info (check cluster_state)
   redis-cli cluster nodes (check for failed nodes)
6. Application fallback: verify app degrades to direct DB queries
7. Emergency recovery: redis-cli FLUSHALL (only if data loss is acceptable)
8. Monitor: redis_exporter for Prometheus, alert on hit_rate < 80%"""
    },
    {
        "id": "rb-006",
        "title": "Disk Space Emergency",
        "team": "platform",
        "content": """When disk usage exceeds 90%:
1. Identify what's using space:
   du -sh /* | sort -rh | head -20
   du -sh /var/log/* | sort -rh | head -10
2. Quick wins — clean up:
   journalctl --vacuum-time=3d (systemd logs)
   docker system prune -f (unused Docker images/containers)
   find /tmp -type f -mtime +7 -delete (old temp files)
3. For Kubernetes nodes:
   crictl rmi --prune (clean unused container images)
   kubectl get pods --all-namespaces -o json | check for completed/failed jobs
4. If /var/log is full: rotate logs
   logrotate -f /etc/logrotate.conf
5. If data partition: identify and archive old data, expand PVC if on cloud
6. Prevention: 
   Alert at 80% with 5-day projection
   Set log rotation policies
   Implement PVC auto-expansion where possible"""
    },
    {
        "id": "rb-007",
        "title": "DNS Resolution Failures",
        "team": "network",
        "content": """When DNS resolution is failing:
1. Test from affected pod:
   kubectl exec -it <pod> -- nslookup kubernetes.default
   kubectl exec -it <pod> -- cat /etc/resolv.conf
2. Check CoreDNS health:
   kubectl get pods -n kube-system -l k8s-app=kube-dns
   kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
3. Check CoreDNS metrics:
   kubectl top pods -n kube-system -l k8s-app=kube-dns
   (DNS pods might be CPU/memory throttled)
4. If CoreDNS is healthy but slow:
   Check ndots setting in pod spec (default 5 causes extra lookups)
   Add FQDN dots to service names: service.namespace.svc.cluster.local.
5. If external DNS failing:
   Check upstream DNS servers in CoreDNS configmap
   kubectl get configmap coredns -n kube-system -o yaml
6. Emergency: add hostAliases to pod spec for critical services"""
    },
    {
        "id": "rb-008",
        "title": "Kubernetes Node NotReady",
        "team": "platform",
        "content": """When a node shows NotReady status:
1. Check node status:
   kubectl describe node <node-name> | grep -A 20 Conditions
   Look for: MemoryPressure, DiskPressure, PIDPressure, NetworkUnavailable
2. SSH into the node (if accessible):
   systemctl status kubelet
   journalctl -u kubelet --since '10 minutes ago'
3. Common causes:
   - Kubelet crashed: systemctl restart kubelet
   - Disk pressure: clean up disk (see Disk Space runbook)
   - Memory pressure: identify and kill memory-heavy processes
   - Network: check node NIC, CNI plugin logs
4. Check if pods were evicted:
   kubectl get pods --all-namespaces --field-selector spec.nodeName=<node>
5. If node unrecoverable:
   kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
   kubectl delete node <node>
   Replace with new node (cloud: terminate instance, ASG creates new one)
6. Prevention: node problem detector, monitoring on kubelet health"""
    }
]

# ============================================================
# STEP 2: Set up ChromaDB and load runbooks
# ============================================================

print("📚 Setting up vector database...")

# Use a persistent database (saved to disk)
chroma_client = chromadb.Client()  # In-memory for speed; use PersistentClient for disk

# Create a collection (like a table in a regular database)
collection = chroma_client.create_collection(
    name="sre_runbooks",
    metadata={"description": "SRE runbook knowledge base for incident response"}
)

# Add all runbooks to the collection
# ChromaDB handles embedding automatically if you don't provide embeddings
# But we'll use our own model for explicit control
embedding_model = SentenceTransformer('all-MiniLM-L6-v2')

for rb in runbooks:
    # Embed the content
    embedding = embedding_model.encode(rb["content"]).tolist()
    
    collection.add(
        ids=[rb["id"]],
        documents=[rb["content"]],
        metadatas=[{"title": rb["title"], "team": rb["team"]}],
        embeddings=[embedding]
    )

print(f"✅ Loaded {collection.count()} runbooks into ChromaDB\n")

# ============================================================
# STEP 3: Search the knowledge base
# ============================================================

def search_runbooks(query: str, n_results: int = 3, team_filter: str = None) -> list:
    """
    Search runbooks using semantic similarity.
    
    Args:
        query: Natural language question
        n_results: How many results to return
        team_filter: Optional — only search docs from this team
    
    Returns:
        List of (title, content, score) tuples
    """
    # Embed the query
    query_embedding = embedding_model.encode(query).tolist()
    
    # Build filter
    where_filter = None
    if team_filter:
        where_filter = {"team": team_filter}
    
    # Search
    results = collection.query(
        query_embeddings=[query_embedding],
        n_results=n_results,
        where=where_filter,
        include=["documents", "metadatas", "distances"]
    )
    
    # Format results
    formatted = []
    for i in range(len(results['ids'][0])):
        formatted.append({
            "id": results['ids'][0][i],
            "title": results['metadatas'][0][i]['title'],
            "team": results['metadatas'][0][i]['team'],
            "distance": round(results['distances'][0][i], 4),
            "content": results['documents'][0][i]
        })
    
    return formatted


# ============================================================
# TEST SEARCHES
# ============================================================

test_queries = [
    ("My pod keeps restarting with OOM errors", None),
    ("Too many database connections", None),
    ("SSL certificate warning", None),
    ("node is showing NotReady", None),
    ("DNS is not working in my pods", None),
    ("disk is almost full", None),
    # Test with team filter
    ("something is slow", "platform"),
    ("cache problems", "data"),
]

print("=" * 70)
print("SEARCH RESULTS")
print("=" * 70)

for query, team in test_queries:
    filter_label = f" [team={team}]" if team else ""
    print(f"\n🔍 Query: \"{query}\"{filter_label}")
    print(f"   {'─' * 60}")
    
    results = search_runbooks(query, n_results=2, team_filter=team)
    
    for i, r in enumerate(results):
        # Lower distance = more similar (ChromaDB uses L2 distance by default)
        relevance = "🟢 HIGH" if r['distance'] < 1.0 else "🟡 MEDIUM" if r['distance'] < 1.5 else "🔴 LOW"
        print(f"   {i+1}. [{relevance}] {r['title']} (distance: {r['distance']})")
        # Show first line of content
        first_line = r['content'].strip().split('\n')[0]
        print(f"      {first_line[:70]}")

# ============================================================
# KEY INSIGHTS
# ============================================================
print(f"\n\n{'=' * 70}")
print("KEY INSIGHTS")
print("=" * 70)
print("""
1. "OOM errors" found the OOMKilled runbook — no keyword overlap needed
2. "SSL certificate" found the TLS runbook — different acronym, same concept
3. "DNS not working" found DNS Resolution runbook — semantic understanding
4. Team filters narrow the search to relevant domain experts
5. Distance score helps rank results — lower = more relevant

This is the RETRIEVAL in RAG. Next week, we:
  - Add document CHUNKING (break big docs into pieces)
  - Add LangChain for pipeline orchestration
  - Connect retrieval to Claude for GENERATION
  
Then your on-call engineer asks a question → system finds the right runbook
→ Claude generates a specific, actionable answer using that runbook.
""")
```

---

### Exercise 3: Measuring Retrieval Quality (10 min)

**Goal:** Build a test suite to measure how accurately your search finds the right documents. This is how you evaluate and improve your RAG system.

Create `week7/retrieval_eval.py`:

```python
"""
Week 7, Exercise 3: Evaluating Retrieval Quality

You need to know: "Is my search actually finding the right documents?"
This test suite tells you.
"""
import chromadb
from sentence_transformers import SentenceTransformer

# Setup (reuse from Exercise 2)
model = SentenceTransformer('all-MiniLM-L6-v2')
client = chromadb.Client()
collection = client.create_collection("eval_test")

# Simplified runbook data for testing
runbook_data = {
    "rb-001": {"text": "Pod CrashLoopBackOff OOMKilled memory limits kubectl logs restart deployment liveness probe image pull error", "title": "Pod CrashLoopBackOff"},
    "rb-002": {"text": "Database connection pool exhausted pg_stat_activity pgbouncer max_connections idle connections PostgreSQL", "title": "DB Connection Pool"},
    "rb-003": {"text": "High latency investigation slow queries Grafana QPS upstream dependencies deployment rollback", "title": "High Latency"},
    "rb-004": {"text": "TLS SSL certificate expiry renewal openssl cert-manager CertificateRequest automatic rotation", "title": "TLS Certificate Expiry"},
    "rb-005": {"text": "Redis cache failure FLUSHALL eviction maxmemory redis-cli cluster hit rate memory", "title": "Redis Cache Failure"},
    "rb-006": {"text": "Disk space full du cleanup log rotation docker prune PVC expansion journalctl vacuum", "title": "Disk Space"},
    "rb-007": {"text": "DNS resolution failure CoreDNS nslookup resolv.conf ndots FQDN upstream nameserver", "title": "DNS Resolution"},
    "rb-008": {"text": "Kubernetes node NotReady kubelet MemoryPressure DiskPressure drain delete CNI network", "title": "Node NotReady"},
}

for rb_id, data in runbook_data.items():
    embedding = model.encode(data["text"]).tolist()
    collection.add(
        ids=[rb_id],
        documents=[data["text"]],
        metadatas=[{"title": data["title"]}],
        embeddings=[embedding]
    )

# ============================================================
# TEST CASES: query → expected runbook ID
# ============================================================
# Each test: what a real on-call engineer might ask → which runbook should be found

test_cases = [
    # Exact match concepts
    ("pod keeps crashing",                    "rb-001", "CrashLoopBackOff"),
    ("container out of memory",               "rb-001", "OOMKilled"),
    ("too many database connections",         "rb-002", "Connection pool"),
    ("postgres connection limit reached",     "rb-002", "max_connections"),
    ("API response times are really high",    "rb-003", "High latency"),
    ("service is slow after deployment",      "rb-003", "Post-deploy slowness"),
    ("SSL certificate warning",               "rb-004", "TLS/SSL"),
    ("cert about to expire",                  "rb-004", "Certificate expiry"),
    ("redis not responding to ping",          "rb-005", "Redis down"),
    ("cache hit rate dropped",                "rb-005", "Cache degradation"),
    ("server disk almost full",               "rb-006", "Disk space"),
    ("no space left on device",               "rb-006", "Disk full"),
    ("DNS lookup failing in pods",            "rb-007", "DNS resolution"),
    ("nslookup not working",                  "rb-007", "DNS tool"),
    ("kubernetes node unresponsive",          "rb-008", "Node NotReady"),
    ("kubelet stopped running",               "rb-008", "Kubelet crash"),
    
    # Harder cases — indirect or vague
    ("payment transactions timing out",       "rb-003", "Latency = timeout"),
    ("can't pull docker image",               "rb-001", "Image pull in CrashLoop"),
    ("liveness probe keeps failing",          "rb-001", "Probe failure"),
    ("application can't connect to postgres", "rb-002", "DB connections"),
]

# ============================================================
# RUN EVALUATION
# ============================================================
print("=" * 70)
print("RETRIEVAL QUALITY EVALUATION")
print("=" * 70)
print(f"\nRunning {len(test_cases)} test queries...\n")

correct_at_1 = 0   # Found in top 1 result
correct_at_3 = 0   # Found in top 3 results
failures = []

for query, expected_id, description in test_cases:
    query_embedding = model.encode(query).tolist()
    results = collection.query(
        query_embeddings=[query_embedding],
        n_results=3
    )
    
    retrieved_ids = results['ids'][0]
    top_1_correct = retrieved_ids[0] == expected_id
    top_3_correct = expected_id in retrieved_ids
    
    if top_1_correct:
        correct_at_1 += 1
    if top_3_correct:
        correct_at_3 += 1
    
    emoji = "✅" if top_1_correct else ("🟡" if top_3_correct else "❌")
    
    if not top_1_correct:
        expected_title = runbook_data[expected_id]["title"]
        got_title = runbook_data[retrieved_ids[0]]["title"]
        failures.append({
            "query": query,
            "expected": f"{expected_id} ({expected_title})",
            "got": f"{retrieved_ids[0]} ({got_title})",
            "in_top_3": top_3_correct
        })
        print(f"  {emoji} \"{query}\"")
        print(f"     Expected: {expected_title} | Got: {got_title}")
    else:
        print(f"  {emoji} \"{query}\" → {runbook_data[expected_id]['title']}")

# ============================================================
# RESULTS SUMMARY
# ============================================================
total = len(test_cases)
precision_at_1 = correct_at_1 / total * 100
precision_at_3 = correct_at_3 / total * 100

print(f"\n{'=' * 70}")
print("RESULTS")
print(f"{'=' * 70}")
print(f"""
  Precision@1: {precision_at_1:.0f}% ({correct_at_1}/{total})
    → Right answer is the #1 result

  Precision@3: {precision_at_3:.0f}% ({correct_at_3}/{total})
    → Right answer is in the top 3 results

  Target for production: Precision@3 > 90%
""")

if failures:
    print(f"  Failed queries ({len(failures)}):")
    for f in failures:
        in_3 = "✅ yes" if f['in_top_3'] else "❌ no"
        print(f"    \"{f['query']}\"")
        print(f"      Expected: {f['expected']}, Got: {f['got']}, In top 3: {in_3}")

print("""
💡 How to improve retrieval quality:
  1. Add more descriptive text to runbooks (richer embeddings)
  2. Try a larger embedding model (all-mpnet-base-v2)
  3. Use multiple embeddings per doc (title + content separately)
  4. Add keyword search as a complement (hybrid search — Week 9)
  5. Fine-tune the embedding model on your domain (advanced)
""")
```

---

## 📝 Drills (20 min)

### Drill 1: Add 3 More Runbooks

Add these to the knowledge base in Exercise 2:
- **API Rate Limiting** — what to do when a service hits rate limits
- **Memory Leak Investigation** — how to find and fix Java/Python memory leaks
- **Kubernetes Network Policy Issues** — pods can't communicate

Write realistic runbook content (at least 8 steps each) and verify they're found by relevant queries.

### Drill 2: Embedding Model Comparison

Modify Exercise 1 to compare two embedding models:
- `all-MiniLM-L6-v2` (384 dimensions, fast)
- `all-mpnet-base-v2` (768 dimensions, better quality)

Run the same similarity pairs with both models. Which one produces better similarity scores for related SRE concepts? Is the quality improvement worth the speed tradeoff?

### Drill 3: Metadata Filtering Deep Dive

Extend the ChromaDB collection with richer metadata:

```python
metadatas=[{
    "title": rb["title"],
    "team": rb["team"],
    "severity_range": "P1-P2",     # What severity this runbook covers
    "last_updated": "2024-10-01",  # When the runbook was last updated
    "category": "kubernetes",       # Category tag
}]
```

Then test filtered searches:
- Find all runbooks for the "platform" team
- Find runbooks tagged with "kubernetes" category
- Find runbooks for P1-P2 severity issues

### Drill 4: Persistent Vector Database

Modify Exercise 2 to use ChromaDB's persistent storage:

```python
# Instead of:
client = chromadb.Client()

# Use:
client = chromadb.PersistentClient(path="./chroma_db")
```

This saves the database to disk. Verify that:
1. Load runbooks once
2. Close the script
3. Open a NEW script that connects to the same path
4. Search works without re-loading the runbooks

This is how production RAG systems work — you load data once, search many times.

---

## ✅ Week 7 Checklist

Before moving to Week 8:

- [ ] Can explain what embeddings are and why they capture meaning
- [ ] Can use SentenceTransformer to embed text locally
- [ ] Can calculate and interpret cosine similarity scores
- [ ] Have a working ChromaDB collection with 8+ runbooks
- [ ] Can search runbooks with natural language queries
- [ ] Can filter search results by metadata (team, category)
- [ ] Can measure retrieval quality (Precision@1, Precision@3)
- [ ] Understand semantic search vs keyword search tradeoffs

---

## 🧠 Key Concepts from This Week

### The RAG Pipeline (we're building this piece by piece)

```
Week 7 (this week):  Documents → Embed → Store → Search
Week 8 (next week):  Documents → Chunk → Embed → Store → Search → Claude → Answer
Week 9 (after):      + Re-ranking, hybrid search, evaluation
Week 10:             + Production concerns, caching, deployment
```

### Embedding Quality = RAG Quality

```
Bad embeddings → wrong documents retrieved → wrong answer from Claude
Good embeddings → right documents retrieved → accurate answer from Claude

The embedding model is the foundation of your entire RAG system.
```

### ChromaDB Quick Reference

```python
# Create client
client = chromadb.Client()                          # In-memory
client = chromadb.PersistentClient(path="./db")     # Persistent

# Create collection
collection = client.create_collection("name")

# Add documents
collection.add(ids=[...], documents=[...], metadatas=[...], embeddings=[...])

# Search
results = collection.query(
    query_embeddings=[vector],
    n_results=3,
    where={"team": "platform"},     # Metadata filter
    include=["documents", "metadatas", "distances"]
)
```

---

*Next week: Building RAG Pipelines — we add document chunking, LangChain integration, and connect retrieval to Claude so your knowledge base can actually ANSWER questions.*
