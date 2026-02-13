# SUPPLEMENT — FILE S2 of 48
## SQL & DATABASES FOR SRE
### PostgreSQL, Indexing, Query Optimization, Redis, ClickHouse, Backup & Recovery
**Week 11 | Apr 26 - Apr 30, 2025 | 5 Days | 1.5 hrs/day**

---

> **WHY DATABASES MATTER FOR SRE**
>
> Every application depends on a database. When the database is slow, EVERYTHING is slow.
> SREs don't write application queries — but they must:
> - Diagnose slow queries that cause incidents
> - Understand indexing to advise developers
> - Monitor replication lag, connection pools, disk usage
> - Perform backup/restore under pressure at 3 AM
> - Manage Redis caching layers
> - Query observability backends (ClickHouse at Visa!)
>
> Apple interview: "Your database response time spiked 10x. Walk me through your investigation."

| Day | Topic | Key Skills |
|-----|-------|-----------|
| 1 | SQL Fundamentals | Data types, CREATE, SELECT, JOIN, GROUP BY, CTEs |
| 2 | PostgreSQL for SRE | pg_stat, connections, locks, vacuum, PgBouncer |
| 3 | Indexing & Query Optimization | B-tree, EXPLAIN ANALYZE, slow query log |
| 4 | Redis & ClickHouse | Caching, persistence, SPL-to-SQL, log analytics |
| 5 | Backup, Replication & Capstone | pg_dump, streaming replication, failover |

---

## DAY 1: SQL FUNDAMENTALS
**Apr 26, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: relational model, data types, DDL/DML
> 00:20-01:10 Hands-on: create tables, write queries, JOINs
> 01:10-01:30 Drills: query challenges

### Relational Databases — What SREs Need to Know

```
A relational database stores data in TABLES (rows and columns).
Tables are related to each other via keys (foreign keys).

Example:
  users table:    id | name       | email             | created_at
                  1  | Siddharth  | sid@visa.com      | 2025-01-15
                  2  | Alice      | alice@visa.com    | 2025-02-01

  orders table:   id | user_id | total  | status    | created_at
                  1  | 1       | 99.99  | completed | 2025-02-10
                  2  | 1       | 49.99  | failed    | 2025-02-11
                  3  | 2       | 29.99  | pending   | 2025-02-12

  user_id in orders points to id in users = FOREIGN KEY relationship
```

### Data Types & Table Creation (DDL)

```sql
-- Common data types:
-- INTEGER / BIGINT       whole numbers
-- NUMERIC(10,2)          exact decimal (money!)
-- VARCHAR(255)           variable-length string
-- TEXT                   unlimited string
-- BOOLEAN               true/false
-- TIMESTAMP             date + time
-- TIMESTAMPTZ           date + time + timezone (ALWAYS use this!)
-- JSONB                 structured JSON (PostgreSQL superpower)
-- UUID                  universally unique ID

-- Create table:
CREATE TABLE servers (
    id          SERIAL PRIMARY KEY,        -- auto-incrementing ID
    hostname    VARCHAR(255) NOT NULL UNIQUE,
    ip_address  VARCHAR(45) NOT NULL,
    environment VARCHAR(20) DEFAULT 'production',
    cpu_cores   INTEGER,
    ram_gb      INTEGER,
    is_active   BOOLEAN DEFAULT true,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE alerts (
    id          SERIAL PRIMARY KEY,
    server_id   INTEGER REFERENCES servers(id),  -- foreign key
    severity    VARCHAR(20) NOT NULL,
    alert_name  VARCHAR(255) NOT NULL,
    message     TEXT,
    resolved    BOOLEAN DEFAULT false,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Insert data:
INSERT INTO servers (hostname, ip_address, environment, cpu_cores, ram_gb)
VALUES ('web-01', '10.0.1.10', 'production', 8, 32);

INSERT INTO servers (hostname, ip_address, environment, cpu_cores, ram_gb)
VALUES
    ('web-02', '10.0.1.11', 'production', 8, 32),
    ('db-01',  '10.0.2.10', 'production', 16, 64),
    ('dev-01', '10.0.3.10', 'staging', 4, 16);

-- Update:
UPDATE servers SET is_active = false WHERE hostname = 'dev-01';
UPDATE servers SET ram_gb = 64, updated_at = NOW() WHERE hostname = 'web-01';

-- Delete:
DELETE FROM alerts WHERE resolved = true AND created_at < NOW() - INTERVAL '90 days';
-- Always use WHERE with DELETE! "DELETE FROM table;" wipes everything.

-- Alter table:
ALTER TABLE servers ADD COLUMN region VARCHAR(20);
ALTER TABLE servers ALTER COLUMN ram_gb SET NOT NULL;
```

### SELECT — Reading Data

```sql
-- Basic:
SELECT * FROM servers;
SELECT hostname, ip_address, environment FROM servers;
SELECT * FROM servers WHERE environment = 'production' AND is_active = true;
SELECT * FROM servers ORDER BY created_at DESC LIMIT 10;

-- Filtering:
SELECT * FROM alerts WHERE severity IN ('critical', 'high');
SELECT * FROM servers WHERE hostname LIKE 'web-%';
SELECT * FROM alerts WHERE created_at > NOW() - INTERVAL '1 hour';
SELECT * FROM alerts WHERE created_at BETWEEN '2025-04-01' AND '2025-04-30';
SELECT * FROM servers WHERE region IS NULL;   -- NULL check (not = NULL!)

-- Aggregation:
SELECT severity, COUNT(*) as count
FROM alerts
GROUP BY severity
ORDER BY count DESC;

-- Count alerts per hour (time-series!):
SELECT DATE_TRUNC('hour', created_at) as hour, COUNT(*) as alerts
FROM alerts
GROUP BY hour
ORDER BY hour;

-- HAVING (filter AFTER grouping):
SELECT server_id, COUNT(*) as alert_count
FROM alerts
WHERE severity = 'critical'
GROUP BY server_id
HAVING COUNT(*) > 5;    -- servers with more than 5 critical alerts
```

### JOINs — Combining Tables

```sql
-- INNER JOIN: only rows that match in BOTH tables
SELECT s.hostname, a.alert_name, a.severity
FROM servers s
INNER JOIN alerts a ON s.id = a.server_id;

-- LEFT JOIN: ALL rows from left table + matching from right (NULL if no match)
SELECT s.hostname, a.alert_name
FROM servers s
LEFT JOIN alerts a ON s.id = a.server_id;
-- Shows ALL servers, even those with zero alerts (alert columns = NULL)

-- This is how you find servers with NO alerts:
SELECT s.hostname
FROM servers s
LEFT JOIN alerts a ON s.id = a.server_id
WHERE a.id IS NULL;

-- JOIN types cheat sheet:
-- INNER JOIN: only matches from both sides
-- LEFT JOIN:  all left + matching right (NULLs where no match)
-- RIGHT JOIN: all right + matching left (rarely used, just flip LEFT)
-- FULL JOIN:  everything from both sides
```

### Subqueries, CTEs & Window Functions

```sql
-- Subquery (query inside a query):
SELECT * FROM servers
WHERE id IN (SELECT server_id FROM alerts WHERE severity = 'critical');

-- CTE (Common Table Expression) — readable alternative to subqueries:
WITH critical_servers AS (
    SELECT server_id, COUNT(*) as critical_count
    FROM alerts
    WHERE severity = 'critical'
    AND created_at > NOW() - INTERVAL '24 hours'
    GROUP BY server_id
)
SELECT s.hostname, cs.critical_count
FROM servers s
JOIN critical_servers cs ON s.id = cs.server_id
WHERE cs.critical_count > 10
ORDER BY cs.critical_count DESC;

-- Window functions (advanced — very useful for SRE dashboards):
-- Calculate moving average (detect gradual degradation):
SELECT
    timestamp,
    response_time,
    AVG(response_time) OVER (
        ORDER BY timestamp
        ROWS BETWEEN 9 PRECEDING AND CURRENT ROW
    ) as moving_avg_10
FROM request_log;

-- Rank servers by alert count:
SELECT
    hostname,
    alert_count,
    RANK() OVER (ORDER BY alert_count DESC) as rank
FROM (
    SELECT s.hostname, COUNT(a.id) as alert_count
    FROM servers s
    LEFT JOIN alerts a ON s.id = a.server_id
    GROUP BY s.hostname
) sub;
```

> **TRANSACTIONS & ISOLATION — INTERVIEW TOPIC**
>
> A transaction groups multiple operations into one atomic unit.
> Either ALL succeed or ALL roll back.
>
> ```sql
> BEGIN;
>   UPDATE accounts SET balance = balance - 100 WHERE id = 1;
>   UPDATE accounts SET balance = balance + 100 WHERE id = 2;
> COMMIT;   -- both happen, or neither
> -- If error: ROLLBACK;
> ```
>
> Isolation levels (from least to most strict):
> - **READ UNCOMMITTED** — can see uncommitted changes (dirty reads)
> - **READ COMMITTED** — PostgreSQL default. Only sees committed data.
> - **REPEATABLE READ** — snapshot at start of transaction
> - **SERIALIZABLE** — full isolation, as if transactions ran one at a time
>
> Interview: "What is a dirty read?" → "Reading data that another transaction hasn't committed yet. If that transaction rolls back, you read data that never existed."

### DAY 1 EXERCISES

1. CREATE TABLE for a monitoring system: servers, alerts, metrics. Add foreign keys.
2. INSERT 10 servers across production/staging environments.
3. SELECT all production servers with more than 8 CPU cores.
4. COUNT alerts per severity level. Which is most common?
5. JOIN servers and alerts: find hostnames with critical alerts in the last 24 hours.
6. LEFT JOIN: find servers that have NEVER had any alerts.
7. CTE: find the top 5 servers by total alert count in the past week.
8. Window function: calculate a running total of daily alert counts.
9. Write a transaction that moves a server from active to inactive and resolves all its alerts.
10. Write a query useful during an incident: error rate per minute for the last hour.

---

## DAY 2: POSTGRESQL FOR SRE
**Apr 27, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: pg_stat views, connection management, VACUUM
> 00:20-01:10 Hands-on: monitoring queries, lock investigation
> 01:10-01:30 Drills: incident simulation

### Connection Monitoring

```sql
-- How many connections? (max_connections default = 100)
SELECT count(*) FROM pg_stat_activity;

-- Connections by state:
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state;
-- active:              running a query right now
-- idle:                connected, doing nothing (connection pool holding it)
-- idle in transaction: IN a transaction but not running query = DANGER!
--   Why danger? Holds locks, blocks VACUUM, wastes a connection slot

-- Who is connected and what are they doing?
SELECT pid, usename, client_addr, state,
       query, now() - query_start as duration
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;

-- Kill a long-running query:
SELECT pg_cancel_backend(pid);      -- gentle (cancel current query)
SELECT pg_terminate_backend(pid);   -- force (kill entire connection)
```

### PgBouncer — Connection Pooling

```
WHY: PostgreSQL forks a new process per connection (~10MB RAM each).
100 connections = 1GB RAM just for connections.
With PgBouncer: 1000 app connections -> 50 actual DB connections.

Modes:
  session:     connection held until client disconnects (safest)
  transaction: connection returned after each transaction (most efficient)
  statement:   connection returned after each statement (most aggressive)

SRE monitoring:
  SHOW POOLS;    -- active, waiting, server connections
  SHOW STATS;    -- queries per second, avg query time
  SHOW CLIENTS;  -- connected clients

Key metrics to alert on:
  - cl_waiting > 0:  clients waiting for a connection = pool exhausted
  - sv_active = sv_max: all server connections in use
  - avg_wait_time increasing: pool is a bottleneck
```

### Locks & Blocking

```sql
-- Find blocked queries (someone is waiting on a lock):
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query,
    now() - blocked.query_start AS wait_duration
FROM pg_stat_activity blocked
JOIN pg_locks bl ON blocked.pid = bl.pid
JOIN pg_locks l ON bl.locktype = l.locktype
    AND bl.relation = l.relation
    AND bl.pid != l.pid
JOIN pg_stat_activity blocking ON l.pid = blocking.pid
WHERE NOT bl.granted;

-- SRE lock response:
-- 1. Identify the blocking query/transaction
-- 2. If it's a long-running txn, contact the team that owns it
-- 3. If it's causing an outage: pg_cancel_backend() first, then pg_terminate_backend()
-- 4. Post-incident: add statement_timeout, fix the application
-- 5. Prevention: SET statement_timeout = '30s'; in application config
```

### VACUUM — Garbage Collection

```sql
-- PostgreSQL uses MVCC: UPDATE creates a NEW row, old row stays as "dead tuple"
-- Without VACUUM: table bloat, slower scans, transaction ID wraparound (catastrophic!)

-- Check vacuum status:
SELECT relname, last_vacuum, last_autovacuum, n_dead_tup, n_live_tup,
       ROUND(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 1) as dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Manual vacuum:
VACUUM VERBOSE tablename;           -- reclaim dead tuples (doesn't lock table)
VACUUM FULL tablename;              -- compact storage (LOCKS TABLE — avoid in prod!)
ANALYZE tablename;                  -- update query planner statistics

-- autovacuum settings (postgresql.conf):
-- autovacuum = on                        # ALWAYS on, never disable
-- autovacuum_vacuum_scale_factor = 0.1   # vacuum when 10% rows are dead
-- autovacuum_analyze_scale_factor = 0.05 # re-analyze when 5% rows changed
-- autovacuum_max_workers = 3             # parallel vacuum workers

-- Transaction ID wraparound — the scariest PostgreSQL failure:
-- Every transaction gets an incrementing 32-bit ID (wraps at ~2 billion)
-- If VACUUM can't reclaim old transaction IDs, PostgreSQL SHUTS DOWN
-- to prevent data corruption.
-- Monitor: SELECT datname, age(datfrozenxid) FROM pg_database;
-- Alert if age > 500,000,000 (half of wraparound threshold)
```

### Key pg_stat Views — Your Monitoring Dashboard

```sql
-- Database-level:
SELECT datname,
    xact_commit, xact_rollback,
    blks_hit, blks_read,
    ROUND(blks_hit::numeric / NULLIF(blks_hit + blks_read, 0) * 100, 2) as cache_hit_pct
FROM pg_stat_database
WHERE datname = current_database();
-- cache_hit_pct should be > 99%. Below 95% = add more shared_buffers or RAM.

-- Table-level:
SELECT relname, seq_scan, idx_scan, n_tup_ins, n_tup_upd, n_tup_del, n_dead_tup
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;
-- High seq_scan on large table with low idx_scan = MISSING INDEX

-- Index usage:
SELECT indexrelname, idx_scan, idx_tup_read, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
-- idx_scan = 0 AND table has many rows = UNUSED index (wasting disk + slowing writes)

-- Table sizes:
SELECT relname,
    pg_size_pretty(pg_total_relation_size(relid)) as total_size,
    pg_size_pretty(pg_relation_size(relid)) as table_size,
    pg_size_pretty(pg_total_relation_size(relid) - pg_relation_size(relid)) as index_size
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
```

### DAY 2 EXERCISES

1. Check connection count and breakdown by state. Any "idle in transaction"?
2. Find the longest-running active query. Is it a problem?
3. Find blocked queries. Who is the blocker? What are they doing?
4. Check vacuum status: top 5 tables by dead tuples. Calculate dead%.
5. Check cache hit ratio. Above 99%? Below = problem.
6. Find tables with most sequential scans. Do they need indexes?
7. Find unused indexes (idx_scan = 0). Total wasted space?
8. Check transaction ID age: `SELECT datname, age(datfrozenxid) FROM pg_database;`
9. Explain PgBouncer transaction mode: why is it better than session mode?
10. Write a "PostgreSQL health check" query that shows connections, cache ratio, dead tuples, table sizes.

---

## DAY 3: INDEXING & QUERY OPTIMIZATION
**Apr 28, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: how indexes work, B-tree internals, EXPLAIN
> 00:20-01:10 Hands-on: create indexes, analyze queries
> 01:10-01:30 Drills: optimization challenges

### How Indexes Work

```sql
-- Without index: SEQUENTIAL SCAN (reads every row, one by one)
-- With index:    INDEX SCAN (B-tree lookup, jumps directly to matching rows)
-- Analogy: book without index = read every page. With index = look up topic, go to page.

-- B-tree index (default, most common):
CREATE INDEX idx_servers_environment ON servers(environment);
CREATE INDEX idx_alerts_server_created ON alerts(server_id, created_at);
-- Composite index: ORDER MATTERS. (server_id, created_at) helps:
--   WHERE server_id = 5                        (yes, uses first column)
--   WHERE server_id = 5 AND created_at > '...' (yes, uses both)
--   WHERE created_at > '...'                   (NO! can't skip first column)

-- When to create an index:
-- 1. Column in WHERE clauses on large tables (>10K rows)
-- 2. Column in JOIN conditions
-- 3. Column in ORDER BY (avoids sorting)
-- Rule: if a query does Seq Scan on a large table, it probably needs an index

-- When NOT to index:
-- Small tables (< 1000 rows, Seq Scan is faster)
-- Low cardinality columns (boolean: only true/false)
-- Heavily written tables (each index slows INSERT/UPDATE/DELETE)

-- Partial index (index only a subset of rows — smaller, faster):
CREATE INDEX idx_alerts_unresolved ON alerts(created_at)
WHERE resolved = false;
-- Only indexes unresolved alerts. Perfect for "show open alerts" queries.

-- Index types:
-- B-tree:  default. Equality and range (=, <, >, BETWEEN, LIKE 'prefix%')
-- Hash:    equality only (=). Rarely used, B-tree is usually better.
-- GIN:     full-text search, JSONB, arrays
-- GiST:    geometric, range types
-- BRIN:    very large tables with naturally ordered data (append-only logs)
```

### EXPLAIN ANALYZE — See What PostgreSQL Actually Does

```sql
-- EXPLAIN shows the plan. EXPLAIN ANALYZE runs it and shows real numbers.
-- ALWAYS use ANALYZE to see actual vs estimated.

EXPLAIN ANALYZE
SELECT s.hostname, COUNT(a.id) as alert_count
FROM servers s
JOIN alerts a ON s.id = a.server_id
WHERE a.severity = 'critical'
AND a.created_at > NOW() - INTERVAL '24 hours'
GROUP BY s.hostname
ORDER BY alert_count DESC
LIMIT 10;

-- Reading the output:
--
-- Seq Scan on alerts (cost=0.00..15234.00 rows=500 width=120)
--   Filter: severity = 'critical' AND created_at > ...
--   actual time=45.3..89.7 rows=12 loops=1
--   Rows Removed by Filter: 99988
--
-- WHAT TO LOOK FOR:
-- 1. "Seq Scan" on large table         = NEEDS INDEX
-- 2. "Rows Removed by Filter" is huge  = index would skip these
-- 3. actual rows vs estimated rows      = if 100x off, run ANALYZE
-- 4. "Sort" with high cost              = index on ORDER BY column helps
-- 5. "Nested Loop" with high loops      = possible N+1 query problem
-- 6. actual time                        = > 100ms for web = too slow

-- After adding index:
CREATE INDEX idx_alerts_sev_created ON alerts(severity, created_at);

-- Re-run EXPLAIN ANALYZE:
-- Index Scan using idx_alerts_sev_created on alerts
--   actual time=0.05..0.12 rows=12 loops=1
-- 89ms -> 0.12ms = 742x faster!
```

### Slow Query Log & pg_stat_statements

```sql
-- Enable in postgresql.conf:
-- log_min_duration_statement = 500    -- log queries taking > 500ms
-- shared_preload_libraries = 'pg_stat_statements'

-- Top queries by TOTAL time (biggest overall impact):
SELECT
    LEFT(query, 80) as query_preview,
    calls,
    ROUND((total_exec_time / 1000)::numeric, 1) as total_seconds,
    ROUND(mean_exec_time::numeric, 1) as avg_ms,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Top queries by AVERAGE time (slowest individual queries):
SELECT
    LEFT(query, 80) as query_preview,
    calls,
    ROUND(mean_exec_time::numeric, 1) as avg_ms,
    ROUND(stddev_exec_time::numeric, 1) as stddev_ms
FROM pg_stat_statements
WHERE calls > 10
ORDER BY mean_exec_time DESC
LIMIT 10;

-- High stddev = inconsistent performance = sometimes fast, sometimes slow
-- Investigate: maybe lock contention or cache misses
```

### DAY 3 EXERCISES

1. Create a table with 1 million rows (use generate_series). Query without index. Note time.
2. Add a B-tree index. Re-query. How much faster?
3. EXPLAIN ANALYZE a slow query. Find the Seq Scan.
4. Create a composite index. EXPLAIN ANALYZE again. Compare plans.
5. Create a partial index for `WHERE resolved = false`. Test with matching query.
6. Enable pg_stat_statements. Find the 5 slowest queries on your database.
7. Find a query where estimated rows is wildly different from actual. Fix with ANALYZE.
8. Interview scenario: "Database is slow, what do you check?"
   Walk through: connections -> locks -> slow queries -> EXPLAIN -> indexes -> vacuum -> disk I/O.

---

## DAY 4: REDIS & CLICKHOUSE
**Apr 29, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: Redis data model, ClickHouse columnar concepts
> 00:20-01:10 Hands-on: Redis commands, ClickHouse queries
> 01:10-01:30 Drills: when to use which database

### Redis — In-Memory Data Store

```bash
# Redis stores ALL data in memory = microsecond reads
# Used for: caching, sessions, rate limiting, queues, leaderboards, pub/sub

redis-cli           # connect to local Redis

# STRING (simplest):
SET user:1001:name "Siddharth"
GET user:1001:name
DEL user:1001:name
SET session:abc123 "user_data" EX 3600    # expires in 1 hour
TTL session:abc123                         # seconds remaining (-1 = no expiry)

# HASH (like a mini row/object):
HSET server:web01 hostname "web-01" ip "10.0.1.10" status "healthy" cpu "45"
HGET server:web01 status
HGETALL server:web01
HINCRBY server:web01 cpu 5               # increment field

# LIST (queue):
LPUSH alerts:queue "disk_full_web01"     # push to left
LPUSH alerts:queue "cpu_high_db01"
RPOP alerts:queue                         # pop from right = FIFO queue
LLEN alerts:queue                         # queue length

# SET (unique members):
SADD tags:web01 "production" "web" "us-east"
SMEMBERS tags:web01
SISMEMBER tags:web01 "production"         # is it in the set?
SINTER tags:web01 tags:web02              # common tags between servers

# SORTED SET (ranked data):
ZADD response_times 23.5 "web01" 45.2 "web02" 12.1 "web03"
ZRANGEBYSCORE response_times 0 30        # servers under 30ms
ZREVRANGE response_times 0 2             # top 3 slowest
```

### Redis Persistence & Eviction

```
PERSISTENCE (how Redis survives restarts):

  RDB (snapshots):
    - Periodic full dump to disk (e.g., every 5 min if 100+ keys changed)
    - Fast restart, but can lose last few minutes of data
    - Good for: caches, non-critical data

  AOF (Append Only File):
    - Logs every write command to disk
    - Can replay to reconstruct data
    - Options: fsync every second (good balance) or every write (safest, slowest)
    - Good for: sessions, queues, anything you can't lose

  Hybrid (RDB + AOF): use both for best of both worlds

EVICTION POLICIES (when maxmemory is hit):
  noeviction:       return error on writes (safest)
  allkeys-lru:      evict least recently used key (most common for caches)
  volatile-lru:     evict LRU but only keys with TTL set
  allkeys-random:   evict random key
  volatile-ttl:     evict keys closest to expiry
```

### Redis SRE Monitoring

```bash
INFO memory         # used_memory, maxmemory, fragmentation_ratio
INFO stats          # keyspace_hits, keyspace_misses, ops/sec
INFO clients        # connected_clients, blocked_clients
INFO replication    # role, connected_slaves, replication offset

SLOWLOG GET 10      # last 10 commands that exceeded slowlog threshold
DBSIZE              # total key count

# Key SRE metrics to alert on:
# used_memory / maxmemory > 90% = running out of memory
# keyspace_hits / (hits + misses) < 95% = cache hit ratio too low
# evicted_keys > 0 = memory pressure, keys being deleted
# connected_clients near maxclients = connection exhaustion
# fragmentation_ratio > 1.5 = memory fragmentation (restart may help)
```

### ClickHouse — Columnar Analytics

```sql
-- ClickHouse: columnar database for ANALYTICS (not OLTP)
-- Perfect for: log analysis, metrics, time-series, dashboards
-- Visa context: migrated from Splunk to ClickHouse

-- Why columnar?
-- Row store (PostgreSQL): reads ALL columns for each row
-- Column store (ClickHouse): reads ONLY columns you SELECT
-- For analytics (few columns, millions of rows), columnar is 10-100x faster

-- Create table:
CREATE TABLE access_log (
    timestamp    DateTime,
    host         LowCardinality(String),    -- few unique values = optimized
    method       LowCardinality(String),
    url          String,
    status       UInt16,
    response_time Float32,
    user_agent   String
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)           -- partition by month
ORDER BY (host, timestamp);                 -- sort order = query optimization

-- MergeTree: the default ClickHouse engine
-- PARTITION BY: how data is split on disk (always include time!)
-- ORDER BY: how data is sorted within partitions (most-filtered columns first)

-- Analytics queries:
-- Requests per minute with p99 latency:
SELECT
    toStartOfMinute(timestamp) as minute,
    count() as requests,
    avg(response_time) as avg_rt,
    quantile(0.99)(response_time) as p99_rt
FROM access_log
WHERE timestamp > now() - INTERVAL 1 HOUR
GROUP BY minute
ORDER BY minute;

-- Top error URLs:
SELECT url, count() as errors
FROM access_log
WHERE status >= 500
AND timestamp > now() - INTERVAL 24 HOUR
GROUP BY url
ORDER BY errors DESC
LIMIT 20;

-- Error rate over time:
SELECT
    toStartOfFiveMinute(timestamp) as ts,
    count() as total,
    countIf(status >= 500) as errors,
    ROUND(errors / total * 100, 2) as error_pct
FROM access_log
WHERE timestamp > now() - INTERVAL 6 HOUR
GROUP BY ts
ORDER BY ts;
```

### SPL to ClickHouse SQL Translation (Visa Migration)

```
Splunk SPL                                    ClickHouse SQL
─────────────────────────────────────         ──────────────────────────────────
index=prod sourcetype=access_log              SELECT * FROM access_log
  status>=500                                   WHERE status >= 500
  | stats count by host                         GROUP BY host
                                                ORDER BY count() DESC

index=prod | timechart span=5m                SELECT toStartOfFiveMinute(timestamp),
  count by status                               status, count()
                                              GROUP BY 1, status ORDER BY 1

index=prod | where response_time > 1000       SELECT * FROM access_log
  | sort -response_time                         WHERE response_time > 1000
  | head 50                                     ORDER BY response_time DESC LIMIT 50

index=prod | stats avg(rt) p99(rt)            SELECT avg(response_time),
  by host                                       quantile(0.99)(response_time)
                                              FROM access_log GROUP BY host

| dedup is essentially                        SELECT DISTINCT or
                                              GROUP BY with any_value()
```

### DAY 4 EXERCISES

1. Redis: SET/GET/DEL keys. Set a key with 60-second TTL. Watch it expire.
2. Redis: create a HASH for server metadata. Update a field. HGETALL.
3. Redis: implement an alert queue with LPUSH/RPOP. Process 5 alerts.
4. Redis: INFO memory — used_memory? INFO stats — hit ratio? evicted_keys?
5. Redis: SLOWLOG GET — any slow commands? What threshold?
6. ClickHouse: write a query for error count per minute in the last hour.
7. ClickHouse: translate this SPL: `index=prod status>=500 | stats count by url | sort -count | head 10`
8. Decision exercise: for each use case, pick the right database:
   - Cache API responses (Redis / PG / CH?)
   - Store user accounts (Redis / PG / CH?)
   - Analyze 1 billion log lines (Redis / PG / CH?)
   - Session storage with 30-min expiry (Redis / PG / CH?)
   - Generate a weekly business report (Redis / PG / CH?)

---

## DAY 5: BACKUP, REPLICATION & CAPSTONE
**Apr 30, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: backup strategies, replication, failover
> 00:20-01:00 Hands-on: pg_dump, restore, replication monitoring
> 01:00-01:30 Capstone: database incident investigation

### PostgreSQL Backup & Recovery

```bash
# LOGICAL BACKUP (pg_dump) — exports SQL or custom format:
pg_dump -Fc mydb > mydb_$(date +%Y%m%d).dump    # custom format (compressed)
pg_dump -Ft mydb > mydb_$(date +%Y%m%d).tar      # tar format
pg_dumpall > all_databases.sql                    # ALL databases + roles

# Restore:
pg_restore -d mydb mydb_20250430.dump             # from custom format
pg_restore -d mydb --clean mydb_20250430.dump     # drop objects first, then restore
psql mydb < backup.sql                            # from plain SQL

# PHYSICAL BACKUP (pg_basebackup) — for large databases:
pg_basebackup -D /backup/base -Ft -z -P           # full binary backup with progress

# POINT-IN-TIME RECOVERY (PITR):
# Requires WAL archiving:
#   archive_mode = on
#   archive_command = 'cp %p /archive/%f'
# Restore to exact timestamp (e.g., "just before the bad deployment"):
#   recovery_target_time = '2025-04-30 14:30:00'

# BACKUP BEST PRACTICES:
# 1. Test restores regularly (untested backup = no backup)
# 2. Store backups off-site (different region/account)
# 3. Monitor backup age (alert if last backup > 24 hours)
# 4. Document restore procedure (you'll be doing this at 3 AM)
# 5. Know your RPO (Recovery Point Objective: max acceptable data loss)
# 6. Know your RTO (Recovery Time Objective: max acceptable downtime)
```

### Streaming Replication

```
Architecture:
  Primary (read-write) ──WAL stream──> Replica (read-only)
                                        - Offloads read queries
                                        - Hot standby for failover
                                        - Can have multiple replicas

Monitoring:
  -- On primary:
  SELECT client_addr, state, sent_lsn, replay_lsn,
         pg_wal_lsn_diff(sent_lsn, replay_lsn) as lag_bytes
  FROM pg_stat_replication;

  -- On replica:
  SELECT now() - pg_last_xact_replay_timestamp() as replay_lag;

  lag_bytes should be near 0. Growing lag = replica falling behind.

SRE alerts:
  - lag_bytes > 1MB:     warning (replica slightly behind)
  - lag_bytes > 100MB:   critical (risk of data loss on failover)
  - replay_lag > 30s:    critical (stale reads from replica)
  - Replica disconnected: page immediately

Failover:
  - Promote replica: pg_ctl promote -D /var/lib/postgresql/data
  - Update application connection strings to point to new primary
  - Set up new replica from the promoted primary
  - Tools: Patroni, repmgr, pg_auto_failover (automate this!)
```

### CAPSTONE: Database Incident Investigation

> **SCENARIO: P99 latency jumped from 20ms to 2000ms. Application team says "the database is slow."**

```
YOUR INVESTIGATION (20 min):

1. CONNECTIONS (2 min)
   SELECT state, count(*) FROM pg_stat_activity GROUP BY state;
   → Near max_connections? Many "idle in transaction"?
   → PgBouncer: SHOW POOLS; — any cl_waiting?

2. LOCKS (2 min)
   Run the blocking query from Day 2.
   → Is a long transaction holding locks? Kill it if outage.

3. SLOW QUERIES (5 min)
   SELECT LEFT(query,80), mean_exec_time, calls
   FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;
   → Any new slow queries? Did a deployment change query patterns?

4. MISSING INDEXES (3 min)
   SELECT relname, seq_scan, idx_scan FROM pg_stat_user_tables
   WHERE seq_scan > 1000 AND idx_scan < seq_scan
   ORDER BY seq_scan DESC;
   → Sequential scans on large tables = missing index.

5. VACUUM & BLOAT (2 min)
   SELECT relname, n_dead_tup, n_live_tup,
     ROUND(n_dead_tup::numeric/NULLIF(n_live_tup,0)*100,1) as dead_pct
   FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 5;
   → Bloated tables slow down queries.

6. CACHE HIT RATIO (1 min)
   SELECT ROUND(blks_hit::numeric/(blks_hit+blks_read)*100,2) as hit_pct
   FROM pg_stat_database WHERE datname = current_database();
   → Below 99%? Not enough shared_buffers or RAM.

7. REPLICATION (1 min)
   SELECT * FROM pg_stat_replication;
   → Replica falling behind? WAL buildup on disk?

8. SYSTEM RESOURCES (2 min)
   iostat -xz 1 3   (high await = disk bottleneck)
   free -h           (swapping = bad for databases)
   top               (PostgreSQL worker pegged at 100% CPU?)

9. RECENT CHANGES (2 min)
   → Was there a deployment? Schema change? Traffic spike?
   → Check application error logs for clues.

RESULT: ROOT CAUSE → IMMEDIATE FIX → DOCUMENT → LONG-TERM PREVENTION
```

---

> **FILE S2 COMPLETE — SQL & DATABASES FOR SRE**
>
> - SQL fundamentals: data types, CREATE TABLE, INSERT/UPDATE/DELETE, SELECT
> - JOINs: INNER, LEFT, RIGHT, FULL — with SRE-relevant examples
> - Subqueries, CTEs, window functions (moving averages, rankings)
> - Transactions & isolation levels (dirty reads, READ COMMITTED)
> - PostgreSQL monitoring: pg_stat_activity, connections, locks
> - PgBouncer: connection pooling, modes, monitoring
> - VACUUM: autovacuum, dead tuples, transaction ID wraparound
> - Indexing: B-tree, composite, partial, EXPLAIN ANALYZE
> - Slow query log, pg_stat_statements
> - Redis: all data types, persistence (RDB/AOF), eviction, monitoring
> - ClickHouse: MergeTree, partitioning, columnar analytics
> - SPL-to-SQL translation (Visa Splunk-to-ClickHouse migration)
> - Backup: pg_dump, pg_basebackup, PITR, RPO/RTO
> - Replication: streaming, lag monitoring, failover, Patroni
> - Capstone: 9-step database incident investigation checklist
>
> Next: File 06 — Python Fundamentals I (Week 12, May 3-7)
