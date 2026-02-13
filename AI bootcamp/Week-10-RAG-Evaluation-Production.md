# Week 10: RAG Evaluation & Production

## 🎯 Learning Objectives

By the end of this week, you will:
- Build a comprehensive RAG evaluation pipeline testing retrieval AND generation
- Implement semantic caching to avoid redundant LLM calls
- Handle document lifecycle — updates, deletions, versioning
- Build a production-ready RAG service with FastAPI
- Know the full checklist for taking a RAG system from prototype to production
- Complete your `sre-knowledge-base` portfolio project

---

## 📖 Theory (20 min)

### The Gap Between Prototype and Production

Your Week 8-9 RAG pipeline works great in a notebook. But production has challenges notebooks don't:

| Prototype | Production |
|-----------|-----------|
| 8 runbooks | 500+ documents that change weekly |
| You test manually | Automated quality monitoring |
| Single user | 50 on-call engineers hitting it simultaneously |
| Response time doesn't matter | P99 < 3 seconds or nobody uses it |
| No cost tracking | $500/month budget cap |
| Documents never change | Runbooks updated after every incident |

### Production RAG Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  DOCUMENT PIPELINE (runs on schedule or trigger)             │
│                                                              │
│  Confluence/Git → Load → Chunk → Embed → Upsert to VectorDB │
│                                              ↓               │
│                                    Track doc versions        │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  QUERY PIPELINE (runs per request)                           │
│                                                              │
│  Question → Cache Check → [HIT] → Return cached answer      │
│                  ↓ [MISS]                                    │
│  Transform → Hybrid Search → Re-rank → Top K                │
│                  ↓                                           │
│  Augment Prompt → Claude → Answer → Cache Store → Return     │
│                  ↓                                           │
│  Log: query, chunks, answer, latency, cost, quality score   │
└──────────────────────────────────────────────────────────────┘
```

### Semantic Caching — Save Money and Time

If someone asks "how do I fix OOMKilled pods" and 5 minutes later another engineer asks "pod keeps running out of memory" — those are basically the same question. Why pay for a new LLM call?

**Semantic cache:** Store (question_embedding, answer) pairs. When a new question arrives, check if any cached question has a very high similarity (>0.95). If yes, return the cached answer.

**Savings in practice:** 30-50% of on-call questions are variations of the same 20 common issues. Caching eliminates redundant LLM calls.

### Document Lifecycle Management

Documents change. Runbooks get updated after incidents. New runbooks get added. Old procedures get deprecated.

You need:
1. **Upsert** — Add or update a document (replace old chunks with new ones)
2. **Delete** — Remove a deprecated runbook entirely
3. **Versioning** — Know which version of a runbook is in the index
4. **Freshness** — Surface more recent documents when relevant
5. **Refresh** — Periodically re-index from source of truth (Confluence, Git)

### Quality Monitoring in Production

You can't manually check every answer. You need automated quality signals:

| Signal | How to Measure | Threshold |
|--------|---------------|-----------|
| Retrieval relevance | Avg distance of top-K chunks | < 1.2 |
| Answer confidence | LLM self-rated confidence | > 0.7 |
| "I don't know" rate | % of questions with no relevant chunks | < 20% |
| User feedback | Thumbs up/down from engineers | > 80% positive |
| Latency | End-to-end response time | P99 < 3s |
| Cost per query | Tokens × pricing | < $0.05 avg |

---

## 🔨 Hands-On (50 min)

### Exercise 1: Semantic Caching System (15 min)

**Goal:** Build a cache that recognizes semantically similar questions and returns cached answers, saving LLM cost and latency.

Create `week10/semantic_cache.py`:

```python
"""
Week 10, Exercise 1: Semantic Caching for RAG

When two questions mean the same thing, reuse the answer.
Saves 30-50% of LLM costs in practice.
"""
import time
import numpy as np
from sentence_transformers import SentenceTransformer
from dataclasses import dataclass, field
from typing import Optional

embedder = SentenceTransformer('all-MiniLM-L6-v2')


@dataclass
class CacheEntry:
    """A cached question-answer pair."""
    question: str
    question_embedding: np.ndarray
    answer: str
    sources: list
    created_at: float
    hit_count: int = 0


class SemanticCache:
    """
    Cache that matches questions by meaning, not exact text.
    
    "pod out of memory" and "container OOMKilled" will match
    because they have similar embeddings.
    """
    
    def __init__(
        self,
        similarity_threshold: float = 0.92,  # How similar questions must be to cache-hit
        max_entries: int = 1000,               # Max cache size
        ttl_seconds: int = 3600,               # Cache entries expire after 1 hour
    ):
        self.threshold = similarity_threshold
        self.max_entries = max_entries
        self.ttl = ttl_seconds
        self.entries: list[CacheEntry] = []
        self.stats = {"hits": 0, "misses": 0, "evictions": 0}
    
    def _cosine_similarity(self, a: np.ndarray, b: np.ndarray) -> float:
        return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))
    
    def _evict_expired(self):
        """Remove expired entries."""
        now = time.time()
        before = len(self.entries)
        self.entries = [e for e in self.entries if (now - e.created_at) < self.ttl]
        self.stats["evictions"] += before - len(self.entries)
    
    def _evict_lru(self):
        """Remove least-recently-used entries if cache is full."""
        if len(self.entries) >= self.max_entries:
            # Sort by hit_count (ascending), then by created_at (oldest first)
            self.entries.sort(key=lambda e: (e.hit_count, e.created_at))
            removed = len(self.entries) - self.max_entries + 1
            self.entries = self.entries[removed:]
            self.stats["evictions"] += removed
    
    def get(self, question: str) -> Optional[dict]:
        """
        Check if a semantically similar question is cached.
        
        Returns {"answer": ..., "sources": ..., "cached_question": ...} or None
        """
        self._evict_expired()
        
        if not self.entries:
            self.stats["misses"] += 1
            return None
        
        query_emb = embedder.encode(question)
        
        best_match = None
        best_score = 0
        
        for entry in self.entries:
            score = self._cosine_similarity(query_emb, entry.question_embedding)
            if score > best_score:
                best_score = score
                best_match = entry
        
        if best_score >= self.threshold and best_match is not None:
            best_match.hit_count += 1
            self.stats["hits"] += 1
            return {
                "answer": best_match.answer,
                "sources": best_match.sources,
                "cached_question": best_match.question,
                "similarity": round(best_score, 4),
                "cache_age_seconds": round(time.time() - best_match.created_at, 1)
            }
        
        self.stats["misses"] += 1
        return None
    
    def put(self, question: str, answer: str, sources: list = None):
        """Store a question-answer pair in the cache."""
        self._evict_lru()
        
        entry = CacheEntry(
            question=question,
            question_embedding=embedder.encode(question),
            answer=answer,
            sources=sources or [],
            created_at=time.time()
        )
        self.entries.append(entry)
    
    def get_stats(self) -> dict:
        """Get cache performance statistics."""
        total = self.stats["hits"] + self.stats["misses"]
        hit_rate = self.stats["hits"] / total if total > 0 else 0
        return {
            "entries": len(self.entries),
            "hits": self.stats["hits"],
            "misses": self.stats["misses"],
            "hit_rate": f"{hit_rate:.1%}",
            "evictions": self.stats["evictions"]
        }
    
    def clear(self):
        """Clear all cache entries."""
        self.entries.clear()
        self.stats = {"hits": 0, "misses": 0, "evictions": 0}


# ============================================================
# TEST: Semantic cache in action
# ============================================================

cache = SemanticCache(similarity_threshold=0.90)

# Simulate first question (cache miss → would call LLM)
print("=" * 70)
print("SEMANTIC CACHE DEMO")
print("=" * 70)

# First query — cache miss
q1 = "How do I fix a pod that keeps getting OOMKilled?"
result = cache.get(q1)
print(f"\n  Q: \"{q1}\"")
print(f"  Cache: {'HIT' if result else 'MISS'}")

# Simulate LLM response and cache it
cache.put(
    question=q1,
    answer="To fix OOMKilled pods: 1) Check current memory usage with kubectl top pods, 2) Increase memory limits in deployment spec, 3) kubectl edit deployment <name> and change resources.limits.memory",
    sources=["Pod CrashLoopBackOff Recovery"]
)
print(f"  → Stored in cache")

# Similar questions — should hit cache
similar_questions = [
    "pod keeps running out of memory",
    "container OOMKilled how to fix",
    "my pod crashes with out of memory error",
    "OOM killed kubernetes pod resolution",
]

# Different question — should miss cache
different_questions = [
    "how do I check Redis health",
    "database connections are maxed out",
]

print(f"\n  Testing similar questions (should HIT):")
for q in similar_questions:
    result = cache.get(q)
    if result:
        print(f"  ✅ HIT (sim={result['similarity']:.3f}) \"{q}\"")
        print(f"     Matched: \"{result['cached_question'][:50]}...\"")
    else:
        print(f"  ❌ MISS \"{q}\"")

print(f"\n  Testing different questions (should MISS):")
for q in different_questions:
    result = cache.get(q)
    if result:
        print(f"  ⚠️  Unexpected HIT (sim={result['similarity']:.3f}) \"{q}\"")
    else:
        print(f"  ✅ MISS (correct) \"{q}\"")

print(f"\n  📊 Cache stats: {cache.get_stats()}")

# Cost savings estimation
print(f"""
\n💡 Cost savings calculation:
  Average cost per RAG query (with Claude): ~$0.01-0.03
  If 40% of queries hit cache: save 40% of LLM costs
  1000 queries/day × $0.02 avg × 0.4 cache rate = $8/day saved
  Monthly savings: ~$240
  
  Plus: cached responses return in <10ms vs 2-3s for LLM calls
""")
```

---

### Exercise 2: Document Lifecycle Management (15 min)

**Goal:** Handle adding, updating, and deleting documents without rebuilding the entire index.

Create `week10/document_manager.py`:

```python
"""
Week 10, Exercise 2: Document Lifecycle Management

Production RAG needs to handle:
- Adding new runbooks
- Updating existing runbooks (re-chunk, re-embed)
- Deleting deprecated runbooks
- Tracking document versions
"""
import chromadb
from sentence_transformers import SentenceTransformer
import hashlib
import time
from typing import Optional

embedder = SentenceTransformer('all-MiniLM-L6-v2')


class DocumentManager:
    """
    Manages the document lifecycle for a RAG knowledge base.
    
    Handles: add, update, delete, version tracking, freshness.
    """
    
    def __init__(self, collection_name: str = "managed_kb"):
        self.chroma = chromadb.Client()
        self.collection = self.chroma.get_or_create_collection(collection_name)
        self.doc_registry = {}  # doc_id → {title, version, hash, chunk_ids, updated_at}
    
    def _content_hash(self, content: str) -> str:
        """Generate a hash of document content to detect changes."""
        return hashlib.md5(content.encode()).hexdigest()[:12]
    
    def _chunk_text(self, text: str, chunk_size: int = 600, overlap: int = 80) -> list[str]:
        """Simple recursive chunker."""
        if len(text) <= chunk_size:
            return [text.strip()] if text.strip() else []
        
        chunks = []
        start = 0
        while start < len(text):
            end = min(start + chunk_size, len(text))
            # Find natural break point
            if end < len(text):
                for sep in ['\n\n', '\n', '. ', ' ']:
                    bp = text.rfind(sep, start, end)
                    if bp > start:
                        end = bp + len(sep)
                        break
            
            chunk = text[start:end].strip()
            if chunk:
                chunks.append(chunk)
            start = end - overlap if end < len(text) else end
            if start <= 0:
                start = end
        
        return chunks
    
    def add_document(self, doc_id: str, title: str, content: str,
                     team: str = "sre", version: str = "1.0") -> dict:
        """
        Add a new document to the knowledge base.
        Chunks, embeds, and stores it. Tracks metadata.
        """
        if doc_id in self.doc_registry:
            return self.update_document(doc_id, title, content, team, version)
        
        content_hash = self._content_hash(content)
        chunks = self._chunk_text(content)
        chunk_ids = []
        
        for i, chunk in enumerate(chunks):
            chunk_id = f"{doc_id}__chunk{i:03d}"
            search_text = f"{title}\n\n{chunk}"
            embedding = embedder.encode(search_text).tolist()
            
            self.collection.add(
                ids=[chunk_id],
                documents=[chunk],
                metadatas=[{
                    "doc_id": doc_id,
                    "title": title,
                    "team": team,
                    "version": version,
                    "chunk_index": i,
                    "total_chunks": len(chunks),
                    "content_hash": content_hash,
                    "updated_at": time.strftime("%Y-%m-%d %H:%M:%S")
                }],
                embeddings=[embedding]
            )
            chunk_ids.append(chunk_id)
        
        self.doc_registry[doc_id] = {
            "title": title,
            "version": version,
            "content_hash": content_hash,
            "chunk_ids": chunk_ids,
            "updated_at": time.time(),
            "team": team
        }
        
        return {"action": "added", "doc_id": doc_id, "chunks": len(chunks)}
    
    def update_document(self, doc_id: str, title: str, content: str,
                        team: str = "sre", version: str = None) -> dict:
        """
        Update an existing document. Only re-indexes if content actually changed.
        """
        new_hash = self._content_hash(content)
        
        if doc_id in self.doc_registry:
            old_hash = self.doc_registry[doc_id]["content_hash"]
            if new_hash == old_hash:
                return {"action": "unchanged", "doc_id": doc_id, "reason": "content identical"}
            
            # Content changed — delete old chunks first
            self.delete_document(doc_id)
        
        # Auto-increment version
        if version is None:
            old_version = self.doc_registry.get(doc_id, {}).get("version", "0.0")
            try:
                major, minor = old_version.split(".")
                version = f"{major}.{int(minor)+1}"
            except (ValueError, AttributeError):
                version = "1.0"
        
        # Re-add with new content
        result = self.add_document(doc_id, title, content, team, version)
        result["action"] = "updated"
        result["new_version"] = version
        result["old_hash"] = old_hash if doc_id in self.doc_registry else None
        return result
    
    def delete_document(self, doc_id: str) -> dict:
        """Remove a document and all its chunks from the knowledge base."""
        if doc_id not in self.doc_registry:
            return {"action": "not_found", "doc_id": doc_id}
        
        chunk_ids = self.doc_registry[doc_id]["chunk_ids"]
        
        if chunk_ids:
            self.collection.delete(ids=chunk_ids)
        
        del self.doc_registry[doc_id]
        return {"action": "deleted", "doc_id": doc_id, "chunks_removed": len(chunk_ids)}
    
    def list_documents(self) -> list[dict]:
        """List all documents in the knowledge base."""
        docs = []
        for doc_id, info in self.doc_registry.items():
            docs.append({
                "doc_id": doc_id,
                "title": info["title"],
                "version": info["version"],
                "team": info["team"],
                "chunks": len(info["chunk_ids"]),
                "updated_at": time.strftime(
                    "%Y-%m-%d %H:%M:%S", time.localtime(info["updated_at"])
                )
            })
        return docs
    
    def get_stats(self) -> dict:
        """Get knowledge base statistics."""
        return {
            "total_documents": len(self.doc_registry),
            "total_chunks": self.collection.count(),
            "teams": list(set(d["team"] for d in self.doc_registry.values())),
        }


# ============================================================
# TEST: Full document lifecycle
# ============================================================

dm = DocumentManager()

print("=" * 70)
print("DOCUMENT LIFECYCLE MANAGEMENT")
print("=" * 70)

# ADD documents
print("\n📥 Adding documents...")

result = dm.add_document(
    doc_id="rb-001",
    title="Pod CrashLoopBackOff Recovery",
    team="platform",
    content="When a pod is in CrashLoopBackOff: check logs with kubectl logs --previous. Check events with kubectl describe pod. Common causes: OOMKilled (increase memory limits), image pull errors (check registry credentials), application crash (check stack traces), liveness probe failure (review probe config and timeouts)."
)
print(f"  {result}")

result = dm.add_document(
    doc_id="rb-002",
    title="Redis Cache Recovery",
    team="data",
    content="When Redis is down: redis-cli ping should return PONG. Check memory with redis-cli info memory. If maxmemory reached, check eviction policy. For clusters: redis-cli cluster info to verify state. Emergency: FLUSHALL if data loss acceptable."
)
print(f"  {result}")

print(f"\n  📊 Stats: {dm.get_stats()}")

# UPDATE a document (content changed)
print(f"\n📝 Updating rb-001 with new content...")
result = dm.update_document(
    doc_id="rb-001",
    title="Pod CrashLoopBackOff Recovery",
    team="platform",
    content="When a pod is in CrashLoopBackOff: check logs with kubectl logs --previous. Check events with kubectl describe pod. Common causes: OOMKilled (increase memory limits to 1Gi), image pull errors (check registry credentials and image tag), application crash (check stack traces and config), liveness probe failure (increase initialDelaySeconds to 30). NEW: Check if recent deployment caused the issue with kubectl rollout history. Rollback with kubectl rollout undo."
)
print(f"  {result}")

# UPDATE same content (should detect no change)
print(f"\n📝 Updating rb-002 with identical content...")
result = dm.update_document(
    doc_id="rb-002",
    title="Redis Cache Recovery",
    team="data",
    content="When Redis is down: redis-cli ping should return PONG. Check memory with redis-cli info memory. If maxmemory reached, check eviction policy. For clusters: redis-cli cluster info to verify state. Emergency: FLUSHALL if data loss acceptable."
)
print(f"  {result}")

# DELETE a document
print(f"\n🗑️  Deleting deprecated runbook...")
result = dm.delete_document("rb-002")
print(f"  {result}")

# LIST all documents
print(f"\n📋 Current documents:")
for doc in dm.list_documents():
    print(f"  [{doc['doc_id']}] {doc['title']} v{doc['version']} ({doc['chunks']} chunks, {doc['team']})")

print(f"\n  📊 Final stats: {dm.get_stats()}")
```

---

### Exercise 3: Production RAG Service with FastAPI (20 min)

**Goal:** Wrap your RAG pipeline in a FastAPI service with all production features: caching, logging, metrics, and health checks.

Create `week10/rag_service.py`:

```python
"""
Week 10, Exercise 3: Production RAG Service

A complete FastAPI service wrapping your RAG pipeline with:
- Semantic caching
- Request logging
- Latency tracking
- Cost tracking
- Health checks
- Document management API

Install: pip install fastapi uvicorn

Run: uvicorn rag_service:app --reload
Test: curl http://localhost:8000/ask -X POST -H "Content-Type: application/json" \
      -d '{"question": "how to fix OOMKilled pods"}'
"""
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
import time
import chromadb
from sentence_transformers import SentenceTransformer
import anthropic
import numpy as np

# ============================================================
# INITIALIZE COMPONENTS
# ============================================================

app = FastAPI(
    title="SRE Knowledge Base API",
    description="RAG-powered Q&A over SRE runbooks",
    version="1.0.0"
)

embedder = SentenceTransformer('all-MiniLM-L6-v2')
chroma = chromadb.Client()
collection = chroma.get_or_create_collection("production_kb")
claude = anthropic.Anthropic()

# Simple in-memory cache and metrics
cache = {}           # embedding_key → {answer, timestamp}
cache_ttl = 3600     # 1 hour
metrics = {
    "total_queries": 0,
    "cache_hits": 0,
    "cache_misses": 0,
    "total_latency_ms": 0,
    "total_input_tokens": 0,
    "total_output_tokens": 0,
    "errors": 0,
}


# ============================================================
# REQUEST/RESPONSE MODELS
# ============================================================

class AskRequest(BaseModel):
    question: str
    n_results: int = 3
    team_filter: Optional[str] = None
    use_cache: bool = True

class AskResponse(BaseModel):
    answer: str
    sources: list[str]
    confidence: str
    cached: bool
    latency_ms: int
    tokens_used: int

class AddDocRequest(BaseModel):
    doc_id: str
    title: str
    content: str
    team: str = "sre"

class HealthResponse(BaseModel):
    status: str
    documents: int
    chunks: int
    cache_size: int
    cache_hit_rate: str


# ============================================================
# HELPER FUNCTIONS
# ============================================================

def get_cache_key(question: str) -> str:
    """Create a cache key from the question embedding."""
    emb = embedder.encode(question)
    # Round to reduce sensitivity — similar questions get similar keys
    rounded = np.round(emb, decimals=2)
    return str(rounded.tobytes()[:32].hex())

def check_semantic_cache(question: str) -> Optional[dict]:
    """Check if a similar question is cached."""
    query_emb = embedder.encode(question)
    now = time.time()
    
    best_match = None
    best_score = 0
    
    for key, entry in list(cache.items()):
        # Evict expired
        if now - entry["timestamp"] > cache_ttl:
            del cache[key]
            continue
        
        score = float(np.dot(query_emb, entry["embedding"]) / 
                      (np.linalg.norm(query_emb) * np.linalg.norm(entry["embedding"])))
        
        if score > best_score:
            best_score = score
            best_match = entry
    
    if best_score > 0.92 and best_match:
        return best_match
    return None


# ============================================================
# API ENDPOINTS
# ============================================================

@app.post("/ask", response_model=AskResponse)
async def ask_question(request: AskRequest):
    """Ask a question to the SRE knowledge base."""
    start = time.time()
    metrics["total_queries"] += 1
    
    try:
        # Check cache
        if request.use_cache:
            cached = check_semantic_cache(request.question)
            if cached:
                metrics["cache_hits"] += 1
                latency = int((time.time() - start) * 1000)
                metrics["total_latency_ms"] += latency
                return AskResponse(
                    answer=cached["answer"],
                    sources=cached["sources"],
                    confidence=cached["confidence"],
                    cached=True,
                    latency_ms=latency,
                    tokens_used=0
                )
        
        metrics["cache_misses"] += 1
        
        # Retrieve
        query_emb = embedder.encode(request.question).tolist()
        where_filter = {"team": request.team_filter} if request.team_filter else None
        
        results = collection.query(
            query_embeddings=[query_emb],
            n_results=request.n_results,
            where=where_filter,
            include=["documents", "metadatas", "distances"]
        )
        
        if not results['documents'][0]:
            latency = int((time.time() - start) * 1000)
            return AskResponse(
                answer="I don't have a runbook covering this topic.",
                sources=[], confidence="none",
                cached=False, latency_ms=latency, tokens_used=0
            )
        
        # Build context
        context_parts = []
        sources = set()
        avg_distance = 0
        
        for i in range(len(results['documents'][0])):
            doc = results['documents'][0][i]
            meta = results['metadatas'][0][i]
            dist = results['distances'][0][i]
            
            context_parts.append(f"[Source: {meta.get('title', 'Unknown')}]\n{doc}")
            sources.add(meta.get('title', 'Unknown'))
            avg_distance += dist
        
        avg_distance /= len(results['documents'][0])
        context = "\n\n---\n\n".join(context_parts)
        
        # Determine confidence
        confidence = "high" if avg_distance < 0.8 else "medium" if avg_distance < 1.3 else "low"
        
        # Generate
        response = claude.messages.create(
            model="claude-sonnet-4-5-20250514",
            max_tokens=1200,
            temperature=0,
            system="You are an SRE assistant. Answer using ONLY the provided context. Cite sources. If unsure, say so.",
            messages=[{
                "role": "user",
                "content": f"<context>\n{context}\n</context>\n\n<question>{request.question}</question>"
            }]
        )
        
        answer = response.content[0].text
        tokens = response.usage.input_tokens + response.usage.output_tokens
        
        metrics["total_input_tokens"] += response.usage.input_tokens
        metrics["total_output_tokens"] += response.usage.output_tokens
        
        # Store in cache
        question_emb = embedder.encode(request.question)
        cache_key = get_cache_key(request.question)
        cache[cache_key] = {
            "answer": answer,
            "sources": list(sources),
            "confidence": confidence,
            "embedding": question_emb,
            "timestamp": time.time()
        }
        
        latency = int((time.time() - start) * 1000)
        metrics["total_latency_ms"] += latency
        
        return AskResponse(
            answer=answer,
            sources=list(sources),
            confidence=confidence,
            cached=False,
            latency_ms=latency,
            tokens_used=tokens
        )
    
    except Exception as e:
        metrics["errors"] += 1
        raise HTTPException(status_code=500, detail=str(e))


@app.post("/documents")
async def add_document(request: AddDocRequest):
    """Add or update a document in the knowledge base."""
    chunks = []
    content = request.content
    chunk_size = 600
    
    # Simple chunking
    start = 0
    while start < len(content):
        end = min(start + chunk_size, len(content))
        if end < len(content):
            bp = content.rfind('\n', start, end)
            if bp > start:
                end = bp + 1
        
        chunk = content[start:end].strip()
        if chunk:
            chunks.append(chunk)
        start = end
    
    # Delete old chunks for this doc
    try:
        existing = collection.get(where={"doc_id": request.doc_id})
        if existing['ids']:
            collection.delete(ids=existing['ids'])
    except Exception:
        pass
    
    # Add new chunks
    for i, chunk in enumerate(chunks):
        chunk_id = f"{request.doc_id}_c{i:03d}"
        emb = embedder.encode(f"{request.title}\n\n{chunk}").tolist()
        collection.add(
            ids=[chunk_id],
            documents=[chunk],
            metadatas=[{
                "doc_id": request.doc_id,
                "title": request.title,
                "team": request.team,
                "chunk_index": i
            }],
            embeddings=[emb]
        )
    
    return {"status": "ok", "doc_id": request.doc_id, "chunks_indexed": len(chunks)}


@app.delete("/documents/{doc_id}")
async def delete_document(doc_id: str):
    """Remove a document from the knowledge base."""
    existing = collection.get(where={"doc_id": doc_id})
    if not existing['ids']:
        raise HTTPException(status_code=404, detail=f"Document {doc_id} not found")
    
    collection.delete(ids=existing['ids'])
    return {"status": "deleted", "doc_id": doc_id, "chunks_removed": len(existing['ids'])}


@app.get("/health", response_model=HealthResponse)
async def health_check():
    """Health check endpoint for monitoring."""
    total = metrics["cache_hits"] + metrics["cache_misses"]
    hit_rate = f"{metrics['cache_hits']/total:.1%}" if total > 0 else "N/A"
    
    return HealthResponse(
        status="healthy",
        documents=collection.count(),
        chunks=collection.count(),
        cache_size=len(cache),
        cache_hit_rate=hit_rate
    )


@app.get("/metrics")
async def get_metrics():
    """Prometheus-style metrics for monitoring dashboards."""
    total_queries = metrics["total_queries"]
    avg_latency = metrics["total_latency_ms"] / total_queries if total_queries > 0 else 0
    
    # Estimate cost
    input_cost = metrics["total_input_tokens"] * 3 / 1_000_000
    output_cost = metrics["total_output_tokens"] * 15 / 1_000_000
    
    return {
        "total_queries": total_queries,
        "cache_hit_rate": metrics["cache_hits"] / total_queries if total_queries > 0 else 0,
        "avg_latency_ms": round(avg_latency),
        "total_tokens": metrics["total_input_tokens"] + metrics["total_output_tokens"],
        "estimated_cost_usd": round(input_cost + output_cost, 4),
        "error_rate": metrics["errors"] / total_queries if total_queries > 0 else 0,
    }


# ============================================================
# STARTUP: Load initial runbooks
# ============================================================

@app.on_event("startup")
async def load_initial_data():
    """Load starter runbooks on service startup."""
    starter_docs = [
        ("rb-001", "Pod CrashLoopBackOff Recovery", "platform",
         "When a pod is in CrashLoopBackOff: check logs with kubectl logs --previous. Check events with kubectl describe pod. OOMKilled: increase memory limits. Image pull errors: check registry credentials. Application crash: check stack traces. Liveness probe failure: adjust probe config and timeouts. Quick restart: kubectl rollout restart deployment. Rollback: kubectl rollout undo deployment."),
        ("rb-002", "Redis Cache Recovery", "data",
         "Redis failure recovery: test with redis-cli ping (expect PONG). Check memory: redis-cli info memory. If full: check maxmemory-policy (should be allkeys-lru). Check clients: redis-cli info clients. For clusters: redis-cli cluster info. Emergency: FLUSHALL if data loss acceptable. Monitor recovery: cache hit rate should return to 90% within 1 hour."),
        ("rb-003", "PostgreSQL Connection Issues", "data",
         "Database connection exhaustion: SELECT count(*), state FROM pg_stat_activity GROUP BY state. Kill idle connections: SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state='idle' AND query_start < now() - interval '10 minutes'. Check max_connections with SHOW max_connections. For recurring issues: deploy pgbouncer as connection pooler."),
    ]
    
    for doc_id, title, team, content in starter_docs:
        # Use the add_document endpoint logic directly
        chunks = [content]  # Simple — one chunk per starter doc
        emb = embedder.encode(f"{title}\n\n{content}").tolist()
        collection.add(
            ids=[f"{doc_id}_c000"],
            documents=[content],
            metadatas=[{"doc_id": doc_id, "title": title, "team": team, "chunk_index": 0}],
            embeddings=[emb]
        )
    
    print(f"✅ Loaded {len(starter_docs)} starter runbooks ({collection.count()} chunks)")


# ============================================================
# RUN INSTRUCTIONS
# ============================================================
if __name__ == "__main__":
    print("""
    To run this service:
    
    1. pip install fastapi uvicorn
    2. uvicorn rag_service:app --reload --port 8000
    
    Then test with:
    
    # Health check
    curl http://localhost:8000/health
    
    # Ask a question
    curl -X POST http://localhost:8000/ask \\
      -H "Content-Type: application/json" \\
      -d '{"question": "how to fix OOMKilled pods"}'
    
    # Add a document
    curl -X POST http://localhost:8000/documents \\
      -H "Content-Type: application/json" \\
      -d '{"doc_id": "rb-new", "title": "New Runbook", "content": "...", "team": "platform"}'
    
    # Check metrics
    curl http://localhost:8000/metrics
    
    # Interactive docs
    open http://localhost:8000/docs
    """)
```

---

## 📝 Drills (20 min)

### Drill 1: Cache Threshold Tuning

Test the semantic cache with different similarity thresholds:
- 0.85 (loose — more hits, risk of wrong matches)
- 0.90 (balanced)
- 0.95 (strict — fewer hits, very accurate matches)
- 0.98 (very strict — almost exact match only)

For each threshold, test with 10 question pairs (5 that should match, 5 that shouldn't). Find the sweet spot where you get high hit rate without false positives.

### Drill 2: Document Freshness Scoring

Extend the Document Manager with a freshness boost:

```python
def search_with_freshness(self, query, n_results=5, freshness_weight=0.1):
    """
    Boost recently updated documents in search results.
    fresher docs get a small score bonus.
    """
    # 1. Normal vector search
    # 2. For each result, calculate days_since_update
    # 3. Apply freshness boost: final_score = similarity + freshness_weight * recency_score
    # 4. Re-sort by final_score
```

This ensures recently updated runbooks (which might have post-incident improvements) rank higher.

### Drill 3: Load Test Your Service

Write a script that sends 50 queries to the FastAPI service and measures:
- Average latency (first run vs cached runs)
- Cache hit rate
- Error rate
- Total cost

```python
import requests
import time

questions = [
    "how to fix OOMKilled pods",
    "pod keeps running out of memory",     # Should cache-hit
    "Redis is not responding",
    "redis cache failure",                  # Should cache-hit
    # ... 46 more questions
]

for q in questions:
    start = time.time()
    resp = requests.post("http://localhost:8000/ask", json={"question": q})
    latency = (time.time() - start) * 1000
    data = resp.json()
    print(f"{'CACHED' if data['cached'] else 'FRESH':>6} | {latency:>6.0f}ms | {q[:50]}")
```

### Drill 4: Assemble Your Portfolio Project

Create `week10/README.md` for your `sre-knowledge-base` portfolio project:

```markdown
# SRE Knowledge Base — RAG-Powered Runbook Q&A

## What It Does
AI-powered Q&A system for SRE runbooks. On-call engineers ask natural language 
questions and get accurate answers with specific commands, sourced from team runbooks.

## Architecture
- Embedding: all-MiniLM-L6-v2 (local, no API needed)
- Vector DB: ChromaDB
- LLM: Claude Sonnet 4.5
- Serving: FastAPI
- Features: semantic caching, hybrid search, re-ranking, document lifecycle

## Key Metrics
- Retrieval Precision@3: XX%
- Average latency: XXms (cached: XXms)
- Cache hit rate: XX%
- Cost per query: $X.XX average
```

---

## ✅ Week 10 Checklist

Before moving to Week 11:

- [ ] Have a working semantic cache that catches similar questions
- [ ] Can add, update, and delete documents without full re-indexing
- [ ] Have a FastAPI service with /ask, /documents, /health, /metrics endpoints
- [ ] Understand production RAG concerns (caching, latency, cost, monitoring)
- [ ] Can estimate and track RAG system costs
- [ ] Have a portfolio-ready `sre-knowledge-base` project

---

## 🧠 Key Concepts from This Week

### Production RAG Checklist

```
✅ Chunking optimized for document type (500-1000 chars for runbooks)
✅ Query transformation for vague questions
✅ Hybrid search (semantic + keyword)
✅ Re-ranking with cross-encoders
✅ Semantic caching (30-50% cost savings)
✅ Document lifecycle (add/update/delete)
✅ Relevance thresholds (no irrelevant chunks)
✅ "I don't know" handling (no hallucination)
✅ Source attribution
✅ Latency monitoring (target: P99 < 3s)
✅ Cost tracking (tokens and dollars)
✅ Health checks
✅ Quality evaluation (Precision, Recall, MRR)
```

### Cost Model for Production RAG

```
Per query cost breakdown:
  Embedding the question:        ~free (local model)
  Vector search:                 ~free (ChromaDB)
  Cross-encoder re-ranking:      ~free (local model)
  Claude generation:             ~$0.01-0.03 (main cost)
  Semantic cache hit:            ~$0.00

1000 queries/day:
  Without cache: $10-30/day ($300-900/month)
  With 40% cache: $6-18/day ($180-540/month)
  With Haiku for simple Q: $3-10/day ($90-300/month)
```

---

## 🏆 Phase 3 Complete!

You've built a complete RAG system from scratch:
- Week 7: Embeddings & vector search
- Week 8: Full RAG pipeline (chunk → embed → store → retrieve → generate)
- Week 9: Advanced patterns (transform, re-rank, hybrid, evaluation)
- Week 10: Production features (caching, lifecycle, FastAPI service)

**Portfolio project:** `sre-knowledge-base` — a production-ready RAG system that answers SRE questions from your team's runbooks.

---

*Next week: MCP Concepts & Architecture — we learn the Model Context Protocol, the standard way to connect Claude to external tools and data sources. This bridges your RAG system and tool use into a unified protocol.*
