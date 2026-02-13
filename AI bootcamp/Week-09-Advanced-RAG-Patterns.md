# Week 9: Advanced RAG Patterns

## 🎯 Learning Objectives

By the end of this week, you will:
- Transform vague user queries into effective search queries using an LLM
- Implement re-ranking with cross-encoders for dramatically better retrieval precision
- Combine keyword search (BM25) with semantic search for hybrid retrieval
- Build a RAG evaluation framework that measures retrieval AND generation quality
- Understand the advanced patterns that separate toy RAG from production RAG

---

## 📖 Theory (20 min)

### Why Basic RAG Falls Short

Your Week 8 pipeline works — but it has real limitations you'll hit in production:

**Problem 1: Vague queries.** An on-call engineer at 3 AM types: "everything is broken" — that's not a good search query. The embedding is too vague to match any specific runbook.

**Problem 2: Wrong ranking.** Vector search returns the K most similar chunks, but similarity ≠ relevance. The top result might be the 3rd most useful chunk while the best one is ranked 5th.

**Problem 3: Keyword misses.** A search for "pg_terminate_backend" won't match well in embedding space because it's a very specific technical term. Keyword search would find it instantly.

**Problem 4: No quality measurement.** How do you know if your RAG is working well? You need metrics.

### The Advanced RAG Stack

```
Basic RAG:    Query → Embed → Search → Top K → Claude → Answer

Advanced RAG: Query → Transform → [Embed Search + Keyword Search]
                                            ↓
                                    Merge & Deduplicate
                                            ↓
                                      Re-rank (cross-encoder)
                                            ↓
                                        Top K (precise)
                                            ↓
                                     Claude → Answer
                                            ↓
                                    Evaluate (metrics)
```

### Pattern 1: Query Transformation

Use Claude to rewrite vague questions into precise search queries BEFORE searching:

```
User: "everything is broken"

Transformed into:
  1. "service outage error rate spike"
  2. "multiple services degraded simultaneously"  
  3. "cascading failure infrastructure issue"
```

Now you search with 3 specific queries and merge the results. Much better retrieval.

### Pattern 2: Re-Ranking with Cross-Encoders

**Bi-encoder** (what we've been using): Embeds query and document separately, then compares vectors. Fast, but less accurate.

**Cross-encoder**: Takes the query AND document TOGETHER as input and produces a relevance score. Much more accurate, but slower.

```
Bi-encoder:    encode("query") → vector1    encode("doc") → vector2    compare(v1, v2)
Cross-encoder: score("query", "doc") → 0.92 relevance
```

**Production pattern:** Use bi-encoder for fast initial retrieval (get 20 candidates), then cross-encoder to re-rank and pick the best 3-5.

### Pattern 3: Hybrid Search

Combine two search methods to get the best of both:

| | Semantic Search (Embeddings) | Keyword Search (BM25) |
|---|---|---|
| Finds "OOM" for "out of memory" | ✅ Yes | ❌ No |
| Finds exact function name "pg_terminate_backend" | ❌ Weak | ✅ Yes |
| Understands context | ✅ Yes | ❌ No |
| Speed | Fast | Very fast |

Hybrid = run BOTH, merge results, get the benefits of each.

### Pattern 4: RAG Evaluation

You need to measure two things independently:

**Retrieval quality:** Did we find the right chunks?
- Precision@K: Of the K chunks retrieved, how many were actually relevant?
- Recall@K: Of all relevant chunks in the database, how many did we find?
- MRR (Mean Reciprocal Rank): How high is the first relevant result?

**Generation quality:** Did Claude use the chunks correctly?
- Faithfulness: Does the answer only use information from the context?
- Relevance: Does the answer actually address the question?
- Completeness: Does the answer cover all relevant information from the context?

---

## 🔨 Hands-On (50 min)

### Exercise 1: Query Transformation (15 min)

**Goal:** Use Claude to rewrite vague queries into precise search queries. This dramatically improves retrieval for real-world questions.

Create `week9/query_transform.py`:

```python
"""
Week 9, Exercise 1: Query Transformation

Vague questions → precise search queries → better retrieval

This is one of the highest-ROI improvements you can make to a RAG system.
"""
import anthropic
import json
import chromadb
from sentence_transformers import SentenceTransformer

client = anthropic.Anthropic()
embedder = SentenceTransformer('all-MiniLM-L6-v2')


def transform_query(original: str, n_queries: int = 3) -> list[str]:
    """
    Transform a vague user question into multiple precise search queries.
    
    Returns the original query PLUS n_queries alternatives.
    This gives you broader coverage when searching.
    """
    response = client.messages.create(
        model="claude-haiku-4-5-20241022",  # Haiku is perfect for this — fast and cheap
        max_tokens=300,
        temperature=0,
        system="""You generate search queries for an SRE runbook knowledge base.
Given a vague user question, generate precise technical search queries that would
find relevant troubleshooting documentation.
Respond with ONLY a JSON array of strings. No explanation.""",
        messages=[{
            "role": "user",
            "content": f"""Original question: "{original}"

Generate {n_queries} precise search queries as a JSON array.
Each query should focus on a different aspect of the problem.
Use specific technical terms that would appear in runbooks."""
        }]
    )
    
    try:
        alternatives = json.loads(response.content[0].text)
    except json.JSONDecodeError:
        # Fallback: extract from text
        text = response.content[0].text
        start = text.find('[')
        end = text.rfind(']') + 1
        alternatives = json.loads(text[start:end])
    
    # Return original + alternatives (original is often still useful)
    return [original] + alternatives


def search_with_transform(
    question: str,
    collection,
    n_results_per_query: int = 3,
    final_k: int = 5
) -> list[dict]:
    """
    Search using query transformation:
    1. Transform question into multiple queries
    2. Search with each query
    3. Merge and deduplicate results
    4. Return top K unique results
    """
    # Step 1: Transform
    queries = transform_query(question)
    print(f"  🔄 Transformed into {len(queries)} queries:")
    for i, q in enumerate(queries):
        prefix = "  ⚪ original" if i == 0 else f"  🟢 alt {i}"
        print(f"     {prefix}: {q}")
    
    # Step 2: Search with each query
    all_results = {}  # id → {doc, metadata, best_distance}
    
    for query in queries:
        query_emb = embedder.encode(query).tolist()
        results = collection.query(
            query_embeddings=[query_emb],
            n_results=n_results_per_query,
            include=["documents", "metadatas", "distances"]
        )
        
        for j in range(len(results['ids'][0])):
            doc_id = results['ids'][0][j]
            distance = results['distances'][0][j]
            
            if doc_id not in all_results or distance < all_results[doc_id]['distance']:
                all_results[doc_id] = {
                    'id': doc_id,
                    'document': results['documents'][0][j],
                    'metadata': results['metadatas'][0][j],
                    'distance': distance,
                    'found_by': query
                }
    
    # Step 3: Sort by distance and return top K
    sorted_results = sorted(all_results.values(), key=lambda x: x['distance'])
    return sorted_results[:final_k]


# ============================================================
# SETUP: Knowledge base with SRE chunks
# ============================================================
chroma = chromadb.Client()
collection = chroma.create_collection("transform_test")

chunks = [
    {"text": "Pod CrashLoopBackOff OOMKilled: check pod logs with kubectl logs --previous. Increase memory limits in deployment spec.", "title": "Pod Recovery", "id": "c01"},
    {"text": "Redis connection refused: redis-cli ping should return PONG. Check memory with redis-cli info memory. Eviction policy allkeys-lru.", "title": "Redis Recovery", "id": "c02"},
    {"text": "PostgreSQL connections exhausted: SELECT count(*) FROM pg_stat_activity. Kill idle with pg_terminate_backend(pid).", "title": "DB Connections", "id": "c03"},
    {"text": "High latency investigation: check upstream dependencies, database slow queries, resource saturation with kubectl top.", "title": "Latency Runbook", "id": "c04"},
    {"text": "DNS resolution failure in pods: check CoreDNS health, nslookup test, ndots optimization, resolv.conf configuration.", "title": "DNS Issues", "id": "c05"},
    {"text": "Disk space emergency: du -sh /*, clean docker images, journalctl vacuum, log rotation, PVC expansion.", "title": "Disk Space", "id": "c06"},
    {"text": "Node NotReady: check kubelet status, MemoryPressure, DiskPressure conditions. Drain and replace if unrecoverable.", "title": "Node Issues", "id": "c07"},
    {"text": "TLS certificate renewal with cert-manager. Check certificate status, ACME challenges, issuer configuration.", "title": "TLS Certs", "id": "c08"},
]

for c in chunks:
    emb = embedder.encode(c["text"]).tolist()
    collection.add(ids=[c["id"]], documents=[c["text"]], metadatas=[{"title": c["title"]}], embeddings=[emb])

# ============================================================
# TEST: Compare basic search vs transformed search
# ============================================================
vague_questions = [
    "everything is broken",
    "customers can't pay",
    "it's really slow today",
    "some kind of networking problem",
    "the server is full",
]

for question in vague_questions:
    print(f"\n{'=' * 70}")
    print(f"❓ Vague question: \"{question}\"")
    print(f"{'=' * 70}")
    
    # Basic search (no transformation)
    basic_emb = embedder.encode(question).tolist()
    basic = collection.query(query_embeddings=[basic_emb], n_results=2, include=["metadatas", "distances"])
    
    print(f"\n  📊 BASIC search results:")
    for i in range(len(basic['ids'][0])):
        print(f"     {basic['metadatas'][0][i]['title']} (dist={basic['distances'][0][i]:.3f})")
    
    # Transformed search
    print(f"\n  📊 TRANSFORMED search results:")
    transformed_results = search_with_transform(question, collection, final_k=3)
    for r in transformed_results:
        print(f"     {r['metadata']['title']} (dist={r['distance']:.3f}) — found by: \"{r['found_by'][:50]}\"")

print(f"""
\n💡 Key insight:
  "everything is broken" → transformed into specific queries like
  "service outage multiple services" and "cascading failure" →
  finds relevant runbooks that the vague query would miss!
""")
```

---

### Exercise 2: Re-Ranking with Cross-Encoders (20 min)

**Goal:** Use a cross-encoder to re-rank initial retrieval results. This dramatically improves precision — the top result is almost always the best one.

```bash
# Install cross-encoder support (already part of sentence-transformers)
pip install sentence-transformers
```

Create `week9/reranking.py`:

```python
"""
Week 9, Exercise 2: Re-Ranking with Cross-Encoders

Two-stage retrieval:
  Stage 1 (fast): Bi-encoder retrieves 10-20 candidates
  Stage 2 (accurate): Cross-encoder re-scores and picks the best 3-5

This is how production RAG systems work at scale.
"""
from sentence_transformers import SentenceTransformer, CrossEncoder
import chromadb
import numpy as np

# Models
bi_encoder = SentenceTransformer('all-MiniLM-L6-v2')        # Fast, for initial retrieval
cross_encoder = CrossEncoder('cross-encoder/ms-marco-MiniLM-L-6-v2')  # Accurate, for re-ranking

print("✅ Models loaded\n")

# ============================================================
# Knowledge base
# ============================================================
chroma = chromadb.Client()
collection = chroma.create_collection("rerank_test")

# More chunks for realistic ranking challenge
chunks = [
    {"id": "c01", "text": "When pod is OOMKilled, increase memory limits in the deployment spec. Use kubectl edit deployment to change resources.limits.memory from 512Mi to 1Gi or higher."},
    {"id": "c02", "text": "To restart a deployment cleanly: kubectl rollout restart deployment/name. Monitor with kubectl rollout status. If rollback needed: kubectl rollout undo deployment/name."},
    {"id": "c03", "text": "PostgreSQL connection pool exhaustion: Run SELECT count(*), state FROM pg_stat_activity GROUP BY state. Kill idle connections with pg_terminate_backend(pid)."},
    {"id": "c04", "text": "For Java heap memory issues specifically, capture a heap dump before restarting: kubectl exec <pod> -- jmap -dump:live,format=b,file=/tmp/heap.bin 1. Analyze with Eclipse MAT."},
    {"id": "c05", "text": "Pod CrashLoopBackOff general troubleshooting: check logs (kubectl logs --previous), check events (kubectl describe pod), check resource limits, check image pull status."},
    {"id": "c06", "text": "Memory leak investigation for Java services: Monitor heap over time in Grafana. If old gen grows linearly, it's a leak. Common causes: unclosed connections, growing caches, static collections."},
    {"id": "c07", "text": "Redis eviction when memory is full: check maxmemory-policy setting. For caches use allkeys-lru. Check used_memory vs maxmemory with redis-cli info memory."},
    {"id": "c08", "text": "Kubernetes resource management: always set both requests and limits. Requests affect scheduling, limits affect OOM kills. QoS classes: Guaranteed, Burstable, BestEffort."},
    {"id": "c09", "text": "Container image pull errors: check image tag exists in registry, verify imagePullSecrets in service account, check registry authentication. Use kubectl get events for details."},
    {"id": "c10", "text": "Node memory pressure: when a node has MemoryPressure condition, kubelet starts evicting pods. Check with kubectl describe node. Fix: increase node memory or reduce pod memory requests."},
]

for c in chunks:
    emb = bi_encoder.encode(c["text"]).tolist()
    collection.add(ids=[c["id"]], documents=[c["text"]], embeddings=[emb])

print(f"Loaded {collection.count()} chunks\n")


def search_with_rerank(query: str, initial_k: int = 8, final_k: int = 3):
    """
    Two-stage retrieval: fast search → accurate re-ranking.
    """
    # STAGE 1: Fast bi-encoder retrieval
    query_emb = bi_encoder.encode(query).tolist()
    results = collection.query(
        query_embeddings=[query_emb],
        n_results=initial_k,
        include=["documents", "distances"]
    )
    
    candidates = results['documents'][0]
    candidate_ids = results['ids'][0]
    initial_distances = results['distances'][0]
    
    print(f"  Stage 1 — Bi-encoder retrieved {len(candidates)} candidates:")
    for i, (cid, dist) in enumerate(zip(candidate_ids, initial_distances)):
        print(f"    {i+1}. [{cid}] dist={dist:.3f} — {candidates[i][:60]}...")
    
    # STAGE 2: Cross-encoder re-ranking
    # Cross-encoder takes (query, document) PAIRS and scores relevance
    pairs = [(query, doc) for doc in candidates]
    cross_scores = cross_encoder.predict(pairs)
    
    # Sort by cross-encoder score (higher = more relevant)
    ranked = sorted(
        zip(candidate_ids, candidates, cross_scores, initial_distances),
        key=lambda x: x[2],
        reverse=True
    )
    
    print(f"\n  Stage 2 — Cross-encoder re-ranked:")
    for i, (cid, doc, score, orig_dist) in enumerate(ranked):
        moved = ""
        orig_rank = candidate_ids.index(cid) + 1
        new_rank = i + 1
        if orig_rank != new_rank:
            direction = "↑" if new_rank < orig_rank else "↓"
            moved = f" ({direction} was #{orig_rank})"
        
        marker = "✅" if i < final_k else "  "
        print(f"    {marker} {new_rank}. [{cid}] score={score:.3f}{moved}")
        print(f"         {doc[:70]}...")
    
    return ranked[:final_k]


# ============================================================
# TEST: Re-ranking changes the order
# ============================================================
test_queries = [
    "pod running out of memory keeps crashing",
    "how to find a Java memory leak",
    "container image can't be pulled from registry",
]

for query in test_queries:
    print(f"\n{'=' * 70}")
    print(f"❓ {query}")
    print(f"{'=' * 70}")
    
    top_results = search_with_rerank(query)

print(f"""
\n{'=' * 70}
WHY RE-RANKING MATTERS
{'=' * 70}

For "pod running out of memory keeps crashing":
  Bi-encoder might rank general "CrashLoopBackOff" (c05) highest
  Cross-encoder correctly promotes specific "OOMKilled" (c01) to #1

For "Java memory leak":
  Bi-encoder finds general memory chunks
  Cross-encoder promotes the Java-specific leak investigation (c06)

Re-ranking is the single biggest quality improvement you can add to RAG.
The cost is small (cross-encoder on 8-10 texts) but the precision gain is huge.

Production recipe:
  1. Bi-encoder retrieves 15-20 candidates (fast, ~10ms)
  2. Cross-encoder re-ranks them (accurate, ~100ms for 20 docs)
  3. Return top 3-5 to Claude
  
Total additional latency: ~100ms. Worth it for dramatically better answers.
""")
```

---

### Exercise 3: Hybrid Search + Full Evaluation (15 min)

**Goal:** Combine keyword search with semantic search, and build a proper evaluation framework.

Create `week9/hybrid_and_eval.py`:

```python
"""
Week 9, Exercise 3: Hybrid Search + RAG Evaluation Framework

Part A: Hybrid search — keyword (BM25) + semantic (embeddings)
Part B: Evaluation framework — measure retrieval + generation quality
"""
import re
import math
from collections import Counter
from sentence_transformers import SentenceTransformer
import chromadb
import anthropic

embedder = SentenceTransformer('all-MiniLM-L6-v2')


# ============================================================
# PART A: Simple BM25 Keyword Search Implementation
# ============================================================

class BM25:
    """
    BM25 keyword search — finds documents with matching terms.
    Complements semantic search by catching exact technical terms.
    """
    
    def __init__(self, k1: float = 1.5, b: float = 0.75):
        self.k1 = k1
        self.b = b
        self.docs = []
        self.doc_ids = []
        self.doc_lens = []
        self.avg_dl = 0
        self.df = {}  # Document frequency per term
        self.tf = []  # Term frequency per document
    
    def _tokenize(self, text: str) -> list[str]:
        """Simple tokenizer — lowercase, split on non-alphanumeric."""
        return re.findall(r'\w+', text.lower())
    
    def index(self, doc_ids: list[str], documents: list[str]):
        """Index a set of documents for BM25 search."""
        self.doc_ids = doc_ids
        self.docs = documents
        
        for doc in documents:
            tokens = self._tokenize(doc)
            self.doc_lens.append(len(tokens))
            
            tf = Counter(tokens)
            self.tf.append(tf)
            
            for term in set(tokens):
                self.df[term] = self.df.get(term, 0) + 1
        
        self.avg_dl = sum(self.doc_lens) / len(self.doc_lens) if self.doc_lens else 0
    
    def search(self, query: str, k: int = 5) -> list[tuple[str, float]]:
        """Search for documents matching the query. Returns (doc_id, score) pairs."""
        query_tokens = self._tokenize(query)
        n = len(self.docs)
        scores = []
        
        for i, doc_tf in enumerate(self.tf):
            score = 0
            dl = self.doc_lens[i]
            
            for term in query_tokens:
                if term not in doc_tf:
                    continue
                
                tf = doc_tf[term]
                df = self.df.get(term, 0)
                
                # IDF
                idf = math.log((n - df + 0.5) / (df + 0.5) + 1)
                
                # TF normalization
                tf_norm = (tf * (self.k1 + 1)) / (tf + self.k1 * (1 - self.b + self.b * dl / self.avg_dl))
                
                score += idf * tf_norm
            
            scores.append((self.doc_ids[i], score))
        
        scores.sort(key=lambda x: x[1], reverse=True)
        return scores[:k]


def hybrid_search(
    query: str,
    collection,
    bm25: BM25,
    chunks_data: dict,
    semantic_k: int = 5,
    keyword_k: int = 5,
    semantic_weight: float = 0.6,
    keyword_weight: float = 0.4,
    final_k: int = 5
) -> list[dict]:
    """
    Hybrid search: combine semantic + keyword results.
    
    Weight controls how much each method contributes:
    - semantic_weight=0.6 + keyword_weight=0.4 is a good default
    - Increase keyword_weight for technical terms (function names, error codes)
    - Increase semantic_weight for natural language questions
    """
    # Semantic search
    query_emb = embedder.encode(query).tolist()
    sem_results = collection.query(
        query_embeddings=[query_emb],
        n_results=semantic_k,
        include=["distances"]
    )
    
    # Keyword search
    kw_results = bm25.search(query, k=keyword_k)
    
    # Normalize scores to 0-1 range
    # Semantic: convert distance to similarity (lower distance = higher similarity)
    sem_scores = {}
    if sem_results['distances'][0]:
        max_dist = max(sem_results['distances'][0]) + 0.001
        for doc_id, dist in zip(sem_results['ids'][0], sem_results['distances'][0]):
            sem_scores[doc_id] = 1 - (dist / max_dist)
    
    # Keyword: normalize by max score
    kw_scores = {}
    if kw_results:
        max_score = max(s for _, s in kw_results) + 0.001
        for doc_id, score in kw_results:
            kw_scores[doc_id] = score / max_score
    
    # Merge with weighted combination
    all_ids = set(sem_scores.keys()) | set(kw_scores.keys())
    merged = []
    
    for doc_id in all_ids:
        sem_s = sem_scores.get(doc_id, 0)
        kw_s = kw_scores.get(doc_id, 0)
        combined = semantic_weight * sem_s + keyword_weight * kw_s
        
        merged.append({
            'id': doc_id,
            'combined_score': round(combined, 4),
            'semantic_score': round(sem_s, 4),
            'keyword_score': round(kw_s, 4),
            'text': chunks_data[doc_id]['text'],
        })
    
    merged.sort(key=lambda x: x['combined_score'], reverse=True)
    return merged[:final_k]


# ============================================================
# PART B: RAG Evaluation Framework
# ============================================================

def evaluate_retrieval(test_cases: list[dict], search_func, k: int = 3) -> dict:
    """
    Evaluate retrieval quality across a test suite.
    
    test_cases format:
    [
        {"query": "...", "relevant_ids": ["c01", "c04"]},
        ...
    ]
    
    Returns metrics: precision@k, recall@k, MRR
    """
    precisions = []
    recalls = []
    reciprocal_ranks = []
    
    for tc in test_cases:
        query = tc["query"]
        expected = set(tc["relevant_ids"])
        
        # Run search
        results = search_func(query, k)
        retrieved_ids = [r['id'] if isinstance(r, dict) else r[0] for r in results]
        retrieved_set = set(retrieved_ids[:k])
        
        # Precision@K: fraction of retrieved that are relevant
        relevant_retrieved = retrieved_set & expected
        precision = len(relevant_retrieved) / k if k > 0 else 0
        precisions.append(precision)
        
        # Recall@K: fraction of all relevant that were retrieved
        recall = len(relevant_retrieved) / len(expected) if expected else 0
        recalls.append(recall)
        
        # MRR: 1/rank of first relevant result
        rr = 0
        for i, rid in enumerate(retrieved_ids):
            if rid in expected:
                rr = 1 / (i + 1)
                break
        reciprocal_ranks.append(rr)
    
    return {
        "precision_at_k": round(sum(precisions) / len(precisions), 3),
        "recall_at_k": round(sum(recalls) / len(recalls), 3),
        "mrr": round(sum(reciprocal_ranks) / len(reciprocal_ranks), 3),
        "num_test_cases": len(test_cases)
    }


# ============================================================
# TEST EVERYTHING
# ============================================================

# Setup knowledge base
chunks_data = {
    "c01": {"text": "Pod OOMKilled: increase memory limits. kubectl edit deployment resources.limits.memory. Check current usage with kubectl top pods."},
    "c02": {"text": "kubectl rollout restart deployment to restart pods cleanly. kubectl rollout undo to rollback to previous version."},
    "c03": {"text": "PostgreSQL connection pool: SELECT count(*) FROM pg_stat_activity. pg_terminate_backend(pid) to kill idle. pgbouncer for connection pooling."},
    "c04": {"text": "Java heap dump: jmap -dump:live,format=b,file=heap.bin. Analyze with Eclipse MAT. Common leak: unclosed connections, static collections."},
    "c05": {"text": "CrashLoopBackOff: check kubectl logs --previous, kubectl describe pod for events, verify resource limits and image pull status."},
    "c06": {"text": "Redis memory: redis-cli info memory. maxmemory-policy allkeys-lru for caches. FLUSHALL for emergency data clear. Monitor with redis_exporter."},
    "c07": {"text": "DNS in Kubernetes: CoreDNS health check, nslookup from pod, resolv.conf ndots setting, FQDN with trailing dot for optimization."},
    "c08": {"text": "Disk space: du -sh /*, docker system prune, journalctl --vacuum-time=3d, logrotate for log management, PVC expansion on cloud."},
}

chroma = chromadb.Client()
collection = chroma.create_collection("hybrid_test")

for cid, data in chunks_data.items():
    emb = embedder.encode(data["text"]).tolist()
    collection.add(ids=[cid], documents=[data["text"]], embeddings=[emb])

# Setup BM25
bm25 = BM25()
bm25.index(list(chunks_data.keys()), [d["text"] for d in chunks_data.values()])

# Hybrid search test
print("=" * 70)
print("HYBRID SEARCH TEST")
print("=" * 70)

test_queries_hybrid = [
    "pg_terminate_backend idle connections",  # Exact term — keyword should help
    "container running out of memory",        # Semantic — embeddings should help
    "jmap heap dump java",                    # Mix of exact terms and concept
]

for query in test_queries_hybrid:
    print(f"\n🔍 \"{query}\"")
    results = hybrid_search(query, collection, bm25, chunks_data, final_k=3)
    for r in results:
        print(f"  [{r['id']}] combined={r['combined_score']:.3f} (sem={r['semantic_score']:.3f}, kw={r['keyword_score']:.3f})")
        print(f"        {r['text'][:70]}...")

# Evaluation test
print(f"\n\n{'=' * 70}")
print("RAG EVALUATION")
print("=" * 70)

eval_test_cases = [
    {"query": "pod out of memory crash",          "relevant_ids": ["c01", "c05"]},
    {"query": "how to find Java memory leak",     "relevant_ids": ["c04"]},
    {"query": "database connections exhausted",   "relevant_ids": ["c03"]},
    {"query": "redis cache full eviction",        "relevant_ids": ["c06"]},
    {"query": "DNS not resolving in pod",         "relevant_ids": ["c07"]},
    {"query": "disk full cleanup",                "relevant_ids": ["c08"]},
    {"query": "rollback deployment",              "relevant_ids": ["c02"]},
    {"query": "pg_terminate_backend kill query",  "relevant_ids": ["c03"]},
]

# Compare semantic-only vs hybrid
def semantic_search_func(query, k):
    emb = embedder.encode(query).tolist()
    r = collection.query(query_embeddings=[emb], n_results=k, include=["distances"])
    return [{'id': rid} for rid in r['ids'][0]]

def hybrid_search_func(query, k):
    return hybrid_search(query, collection, bm25, chunks_data, final_k=k)

sem_metrics = evaluate_retrieval(eval_test_cases, semantic_search_func, k=3)
hyb_metrics = evaluate_retrieval(eval_test_cases, hybrid_search_func, k=3)

print(f"\n  {'Metric':<20} {'Semantic Only':>15} {'Hybrid':>15}")
print(f"  {'─'*20} {'─'*15} {'─'*15}")
print(f"  {'Precision@3':<20} {sem_metrics['precision_at_k']:>15.3f} {hyb_metrics['precision_at_k']:>15.3f}")
print(f"  {'Recall@3':<20} {sem_metrics['recall_at_k']:>15.3f} {hyb_metrics['recall_at_k']:>15.3f}")
print(f"  {'MRR':<20} {sem_metrics['mrr']:>15.3f} {hyb_metrics['mrr']:>15.3f}")

print(f"""
\n💡 Hybrid search should improve on keyword-heavy queries like "pg_terminate_backend"
   where semantic search alone might miss the exact function name.

   The evaluation framework lets you:
   1. Measure current quality objectively
   2. Test changes (new chunking, different models) and see if metrics improve
   3. Set quality thresholds for production (e.g., MRR > 0.8)
   4. Catch quality regressions when you add new documents
""")
```

---

## 📝 Drills (20 min)

### Drill 1: Query Transformation + Re-Ranking Pipeline

Combine Exercise 1 and Exercise 2 into a single pipeline:

```python
def advanced_rag_search(question, collection):
    # Step 1: Transform query (get 3 alternatives)
    # Step 2: Search with all queries (bi-encoder, get 15 candidates)
    # Step 3: Deduplicate
    # Step 4: Re-rank with cross-encoder
    # Step 5: Return top 5
    pass
```

Test with the vague questions from Exercise 1. Does the combination of transformation + re-ranking give better results than either alone?

### Drill 2: Tune Hybrid Search Weights

Run the evaluation from Exercise 3 with different weight combinations:

```
semantic=1.0, keyword=0.0  (pure semantic)
semantic=0.8, keyword=0.2
semantic=0.6, keyword=0.4  (default)
semantic=0.4, keyword=0.6
semantic=0.0, keyword=1.0  (pure keyword)
```

Which weight combination gives the best MRR? Does it depend on the type of query?

### Drill 3: End-to-End RAG Evaluation

Build a complete eval that measures BOTH retrieval AND generation:

```python
eval_cases = [
    {
        "query": "pod out of memory",
        "relevant_ids": ["c01"],
        "expected_answer_contains": ["kubectl edit", "memory limits", "resources.limits.memory"],
        "expected_answer_excludes": ["Redis", "database", "DNS"]  # Shouldn't mention irrelevant topics
    }
]
```

For generation quality, check:
- Does Claude's answer contain the expected keywords?
- Does Claude's answer avoid mentioning irrelevant topics?
- Does Claude cite the source?

### Drill 4: Add to Your SREKnowledgeBase

Go back to your Week 8 `SREKnowledgeBase` class and add:

1. **Query transformation** in the `ask()` method (before search)
2. **Re-ranking** after initial retrieval (before sending to Claude)
3. **Evaluation method** that runs your test suite and reports metrics

This upgraded knowledge base is your `sre-knowledge-base` portfolio project!

---

## ✅ Week 9 Checklist

Before moving to Week 10:

- [ ] Can transform vague queries into precise search queries
- [ ] Can implement re-ranking with cross-encoders
- [ ] Understand how hybrid search combines keyword + semantic
- [ ] Have a working BM25 implementation
- [ ] Can evaluate retrieval quality (Precision, Recall, MRR)
- [ ] Understand the advanced RAG stack (transform → search → merge → re-rank)
- [ ] Can measure the impact of each improvement objectively

---

## 🧠 Key Concepts from This Week

### The Advanced RAG Recipe

```
1. Query Transformation    — Turn vague questions into precise searches
2. Multi-Query Search      — Search with multiple reformulated queries
3. Hybrid Retrieval        — Combine semantic + keyword for best coverage
4. Cross-Encoder Re-Rank   — Promote the most relevant results to the top
5. Quality Evaluation      — Measure everything, improve systematically
```

### When to Use Each Pattern

```
Query Transformation: When users ask vague or conversational questions
Re-Ranking:          Always — cheap and always improves precision
Hybrid Search:       When your docs contain specific technical terms
                     (function names, error codes, CLI commands)
Evaluation:          Always — you can't improve what you can't measure
```

### Cost vs Quality Tradeoffs

```
Pattern                 Added Latency    Quality Impact
Query Transform         ~200ms           +15-25% retrieval recall
Re-Ranking (20 docs)    ~100ms           +20-30% precision
Hybrid Search           ~50ms            +10-15% for keyword queries
All combined            ~350ms           Significantly better answers
```

---

*Next week: RAG Evaluation & Production — caching, performance optimization, deployment patterns, and building the production-ready version of your SRE knowledge base.*
