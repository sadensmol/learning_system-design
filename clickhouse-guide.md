# ClickHouse: Complete Guide

> Everything you need to know about ClickHouse — what it is, why it's absurdly fast, how to run it, how to drive it from Go, and how quants/HFT/algo-trading teams actually use it in production. Plus hidden gems nobody tells you about.

---

## Table of Contents

1. [What is ClickHouse?](#1-what-is-clickhouse)
2. [The Mental Model](#2-the-mental-model)
3. [Why is it So Fast?](#3-why-is-it-so-fast)
4. [Architecture Deep Dive](#4-architecture-deep-dive)
5. [Table Engines (The Heart of ClickHouse)](#5-table-engines-the-heart-of-clickhouse)
6. [MergeTree in Detail](#6-mergetree-in-detail)
7. [Data Types You Should Actually Use](#7-data-types-you-should-actually-use)
8. [Installation and Setup](#8-installation-and-setup)
9. [Tools for Working with ClickHouse](#9-tools-for-working-with-clickhouse)
10. [Using ClickHouse from Go](#10-using-clickhouse-from-go)
11. [Where ClickHouse Shines (and Where it Doesn't)](#11-where-clickhouse-shines-and-where-it-doesnt)
12. [HFT, Algo Trading, Quant Analytics](#12-hft-algo-trading-quant-analytics)
13. [Backtester Patterns](#13-backtester-patterns)
14. [ClickHouse vs The World](#14-clickhouse-vs-the-world)
15. [Is ClickHouse Still Relevant in 2026?](#15-is-clickhouse-still-relevant-in-2026)
16. [Fintech & Trading Competitors — Detailed Comparison](#16-fintech--trading-competitors--detailed-comparison)
17. [Best Practices](#17-best-practices)
18. [Hidden Gems & Deep Cuts](#18-hidden-gems--deep-cuts)
19. [Common Pitfalls](#19-common-pitfalls)
20. [Operations and Monitoring](#20-operations-and-monitoring)
21. [When NOT to Use ClickHouse](#21-when-not-to-use-clickhouse)
22. [Further Reading](#22-further-reading)

---

## 1. What is ClickHouse?

**ClickHouse** is an open-source **columnar OLAP database management system** built for running analytical queries on massive datasets in real time. It was created at Yandex around 2009 for web analytics (originally for Yandex.Metrica, processing ~20 billion events per day) and open-sourced in 2016.

The short version: **it stores data by column, compresses it aggressively, and vectorizes execution**. The result is query performance that is frequently 100× to 1000× faster than row-stores like PostgreSQL or MySQL for analytical workloads — while still handling billions of rows per second on a single machine.

```
Row Store (Postgres, MySQL):          Column Store (ClickHouse):
┌──┬──────┬───────┬─────┐              Col A: [1, 2, 3, 4, ...]
│id│ name │ price │ ts  │              Col B: ["A","B","C","D",...]
├──┼──────┼───────┼─────┤              Col C: [10.1, 20.2, 30.3,..]
│ 1│ BTC  │ 50000 │ ... │              Col D: [t1, t2, t3, t4, ...]
│ 2│ ETH  │  3000 │ ... │
│ 3│ SOL  │   100 │ ... │              Reads only the columns you query.
└──┴──────┴───────┴─────┘              Compresses similar values together.
```

### Key characteristics

- **Columnar storage** — each column stored as a separate file, compressed independently
- **Vectorized execution** — operates on batches of column values, not row-by-row
- **SQL-compatible** — a superset of standard SQL with powerful extensions
- **Distributed by design** — sharding, replication, and parallel query execution built in
- **Append-optimized** — loves bulk inserts, hates small transactional writes
- **Real-time analytics** — you can query data milliseconds after it was ingested

---

## 2. The Mental Model

Before going deeper, internalize this:

> **ClickHouse is not a replacement for your transactional database. It's the query engine you bolt onto it.**

Your Postgres/MySQL/Mongo holds the source of truth with ACID transactions. ClickHouse holds a denormalized, append-only copy optimized for scanning billions of rows in milliseconds. You stream events into ClickHouse (via Kafka, CDC, batch loads) and hammer it with analytical queries.

```
      ┌─────────────┐   writes    ┌────────────┐
      │ Application │ ──────────> │  Postgres  │   (OLTP, source of truth)
      └─────────────┘             └──────┬─────┘
            │                            │ CDC / Kafka / batch
            │ events                     ▼
            └────────────────────> ┌────────────┐
                                   │ ClickHouse │   (OLAP, analytics)
                                   └────────────┘
                                          ▲
                                          │  SELECT ... GROUP BY ...
                                   ┌──────┴─────┐
                                   │ Dashboards │
                                   │  BI / ML   │
                                   └────────────┘
```

---

## 3. Why is it So Fast?

ClickHouse's speed isn't one trick — it's the *compounding* of about a dozen tricks. Each gives ~2-5× improvement; together they produce orders of magnitude.

### 3.1 Columnar storage

When you query `SELECT avg(price) FROM trades`, a row store has to read the entire row (id, symbol, price, volume, timestamp, exchange, ...) and discard 95% of it. ClickHouse reads only the `price` column file. That alone is often a 10-20× win on wide tables.

### 3.2 Aggressive compression

Adjacent values in a column are similar, so they compress extremely well. A `Date` column with a trillion rows may compress 100:1. Default codec is **LZ4** (fast), but you can stack codecs:

```sql
CREATE TABLE trades (
    ts        DateTime64(9) CODEC(Delta, ZSTD(3)),
    symbol    LowCardinality(String),
    price     Float64       CODEC(Gorilla, LZ4),
    volume    UInt32        CODEC(T64, LZ4)
) ENGINE = MergeTree ORDER BY (symbol, ts);
```

- **Delta** — stores differences between consecutive values (perfect for timestamps)
- **DoubleDelta** — delta of delta (perfect for monotonically growing values)
- **Gorilla** — Facebook's float compression (perfect for prices, metrics)
- **T64** — efficient for integer columns with small ranges
- **ZSTD** — higher compression ratio, slower than LZ4

**Hidden gem:** stacking `Delta + ZSTD` on timestamps often compresses to ~1-2 bytes per value.

### 3.3 Vectorized query execution

Instead of processing one row at a time, ClickHouse processes **chunks of 65,536 values** at a time. These chunks fit in L1/L2 CPU cache. The inner loops are then auto-vectorized by the compiler into SIMD instructions (SSE, AVX2, AVX-512).

```
Row-by-row (traditional):               Vectorized (ClickHouse):
for each row:                           for each chunk of 65536 rows:
    read row                                load column chunk into cache
    apply filter                            apply filter (SIMD)
    compute                                 compute (SIMD)
    output                                  output chunk
```

One SIMD instruction can add 8 floats in parallel. On AVX-512, 16 floats per instruction.

### 3.4 Sparse primary index

ClickHouse doesn't index every row. It marks one row per "granule" (default 8,192 rows) in a tiny index that fits in RAM. A trillion-row table has only ~120 million index entries — small enough to keep entirely in memory.

### 3.5 Data skipping indexes

On top of the primary index, you can add **secondary skip indexes**: min/max, bloom filters, set membership. These let ClickHouse skip entire blocks without reading them.

```sql
ALTER TABLE trades ADD INDEX idx_sym symbol TYPE bloom_filter GRANULARITY 4;
```

### 3.6 No random I/O

Data on disk is sorted by the table's `ORDER BY` key. Queries that filter on the prefix of this key read contiguous byte ranges — pure sequential I/O, the fastest thing a disk can do.

### 3.7 Parallel everything

Every query shards across all CPU cores by default. `max_threads` defaults to the number of physical cores. Aggregations run in parallel per thread and merge at the end.

### 3.8 Late materialization

ClickHouse delays reading columns until it absolutely has to. If a `WHERE` filter cuts 99% of rows, the remaining columns are read only for the surviving rows.

### 3.9 JIT compilation (optional)

For complex expressions, ClickHouse can compile them to native machine code on the fly using LLVM (`compile_expressions = 1`).

### 3.10 No transactions, no MVCC overhead

There's no row-level versioning tax. Inserts are immutable blocks; "updates" are background merges. This removes gigantic amounts of bookkeeping that traditional DBs carry.

---

## 4. Architecture Deep Dive

### 4.1 The physical layout

```
/var/lib/clickhouse/data/<database>/<table>/
├── all_1_1_0/                    ← a "part" (immutable data block)
│   ├── checksums.txt
│   ├── columns.txt
│   ├── count.txt
│   ├── primary.idx               ← sparse primary index
│   ├── ts.bin                    ← column data (compressed)
│   ├── ts.mrk2                   ← marks file (offsets into .bin)
│   ├── symbol.bin
│   ├── symbol.mrk2
│   ├── price.bin
│   └── price.mrk2
├── all_2_2_0/                    ← another part (from another insert)
└── all_1_2_1/                    ← merged part (level 1)
```

**Parts** are immutable. Every `INSERT` creates a new part. A background thread periodically **merges** parts into bigger parts. Old parts get deleted after ~8 minutes.

This design means:
- Inserts never block reads
- Writes are always sequential appends
- "Compaction" is just file concatenation (very cheap)

### 4.2 Granules and marks

Each column `.bin` file is logically divided into **granules** (default 8,192 rows). The `.mrk2` file stores offsets: "granule N starts at byte X in the compressed file, byte Y in the uncompressed stream."

When reading, ClickHouse:
1. Uses the primary index to find which granules might match
2. Reads marks to get byte offsets
3. Reads only those compressed byte ranges

### 4.3 Clusters, shards, and replicas

```
┌────────────────── ClickHouse Cluster ──────────────────┐
│                                                         │
│  ┌── Shard 1 ───┐  ┌── Shard 2 ───┐  ┌── Shard 3 ───┐ │
│  │ Replica 1-A  │  │ Replica 2-A  │  │ Replica 3-A  │ │
│  │ Replica 1-B  │  │ Replica 2-B  │  │ Replica 3-B  │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
│                                                         │
│         ↑ Coordination via ClickHouse Keeper            │
│           (formerly ZooKeeper)                          │
└─────────────────────────────────────────────────────────┘
```

- **Shard** — a horizontal partition of the data (different rows on different servers)
- **Replica** — a full copy of a shard (for HA)
- **Distributed table** — a virtual table that fans queries out to all shards

---

## 5. Table Engines (The Heart of ClickHouse)

Unlike Postgres, where tables all behave the same, ClickHouse has **~20 table engines** that radically change semantics. Pick the right one and your life is easy; pick the wrong one and you'll fight it forever.

| Engine | Use Case |
|---|---|
| **MergeTree** | Default. Large datasets, time-series, analytics. |
| **ReplacingMergeTree** | Dedup on merge by a version column. |
| **SummingMergeTree** | Auto-sum numeric columns during merges. |
| **AggregatingMergeTree** | Store intermediate aggregate states. |
| **CollapsingMergeTree** | Undo rows with a `Sign` column (±1). |
| **VersionedCollapsingMergeTree** | Like Collapsing but with versioning (works with out-of-order inserts). |
| **ReplicatedMergeTree** | Any of the above, replicated across nodes. |
| **Distributed** | Virtual table over a cluster. |
| **Kafka** | Read from Kafka as if it were a table. |
| **S3 / URL / File** | Query external data in place. |
| **Memory** | In-memory, non-persistent (lightning fast, volatile). |
| **Buffer** | In-RAM buffer in front of a real table (smooths tiny inserts). |
| **Dictionary** | Wrap an external dictionary as a table. |
| **Join** | Precomputed right-hand side of a JOIN. |
| **MaterializedView** | Automatically transformed copy of another table. |
| **Log / TinyLog** | Tiny append-only datasets (<1M rows). |

The rule of thumb: **if in doubt, use MergeTree.**

---

## 6. MergeTree in Detail

```sql
CREATE TABLE trades (
    ts        DateTime64(9),
    symbol    LowCardinality(String),
    price     Float64,
    volume    UInt32,
    exchange  LowCardinality(String)
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(ts)
ORDER BY (symbol, ts)
SETTINGS index_granularity = 8192;
```

### 6.1 `ORDER BY` is critical

The `ORDER BY` clause defines:
1. How data is physically sorted on disk
2. The primary key (unless you override with `PRIMARY KEY`)
3. Which filters get fast

**Rule:** put the lowest-cardinality column first, highest-cardinality last. `(symbol, ts)` is great because `symbol` has thousands of values and `ts` has billions. Queries filtering by symbol (and optionally by time) hit contiguous data.

### 6.2 `PARTITION BY` — use sparingly

Partitions exist for **data lifecycle management**, not query speed. Typical pattern: `PARTITION BY toYYYYMM(ts)` so you can `DROP PARTITION '202401'` to delete old data instantly.

**Pitfall:** fine-grained partitions (e.g., per-day for 10 years) create thousands of parts and tank merge performance. Rule of thumb: **keep total partitions under ~1,000 per table.**

### 6.3 `PRIMARY KEY` vs `ORDER BY`

By default they're the same. You can make `PRIMARY KEY` a prefix of `ORDER BY` when you want fine-grained sorting but coarse indexing:

```sql
ORDER BY   (symbol, exchange, ts)
PRIMARY KEY (symbol)   -- index only on symbol to save RAM
```

### 6.4 TTL — automatic data lifecycle

```sql
CREATE TABLE logs (
    ts DateTime,
    level String,
    msg  String
)
ENGINE = MergeTree
ORDER BY ts
TTL ts + INTERVAL 30 DAY DELETE,
    ts + INTERVAL 7  DAY TO VOLUME 'cold',
    ts + INTERVAL 1  DAY RECOMPRESS CODEC(ZSTD(17));
```

This single feature replaces cron jobs, archival pipelines, and half of ElasticSearch's ILM.

---

## 7. Data Types You Should Actually Use

### 7.1 LowCardinality — the free lunch

Wrap any string with up to ~10k distinct values in `LowCardinality(String)`. Internally ClickHouse builds a dictionary and stores the data as tiny integer indexes. Queries become 5-10× faster and storage drops 5-20×.

```sql
symbol    LowCardinality(String)   -- 'BTCUSDT', 'ETHUSDT', ...  ✓
exchange  LowCardinality(String)   -- 'BINANCE', 'KRAKEN', ...   ✓
user_id   LowCardinality(String)   -- millions of unique IDs ✗
```

### 7.2 Nullable — avoid if possible

`Nullable(T)` adds a separate bitmap column for null flags. Queries get slower. Prefer sentinel values (0, -1, epoch=0) where appropriate.

### 7.3 Native high-precision types

- `DateTime64(9)` — nanosecond-precision timestamps (essential for HFT)
- `Decimal(38, 18)` — arbitrary precision for money
- `UInt256`, `Int128` — huge integers
- `UUID`, `IPv4`, `IPv6` — with native storage and functions
- `Array(T)`, `Map(K,V)`, `Tuple(...)`, `Nested(...)` — rich composite types

### 7.4 Aggregate function types

```sql
quantiles_state  AggregateFunction(quantiles(0.5, 0.95, 0.99), Float64)
```

This column stores the **intermediate state** of an aggregate. You can precompute it in a materialized view and roll up further at query time. Essential for pre-aggregated metrics.

---

## 8. Installation and Setup

### 8.1 Quick start with Docker

```bash
docker run -d --name clickhouse \
  -p 8123:8123 \
  -p 9000:9000 \
  -v clickhouse_data:/var/lib/clickhouse \
  -e CLICKHOUSE_USER=default \
  -e CLICKHOUSE_PASSWORD=secret \
  clickhouse/clickhouse-server:latest

# Connect
docker exec -it clickhouse clickhouse-client --password secret
```

Ports:
- **8123** — HTTP interface (curl, REST clients)
- **9000** — native TCP protocol (fastest, used by clickhouse-client and Go driver)
- **9009** — inter-server replication
- **9440** — native TLS
- **8443** — HTTPS

### 8.2 macOS (local development)

```bash
brew install --cask clickhouse       # installs `clickhouse` binary
clickhouse server                    # start server
clickhouse client                    # connect (in another shell)
```

For a one-off query without running a server, use `clickhouse local` (see §8.3).

### 8.3 clickhouse-local — SQL over files, no server

This is the **killer hidden tool**. `clickhouse-local` runs ClickHouse's engine as a CLI over local files. Think "SQL-powered awk on steroids."

```bash
clickhouse local --query "
  SELECT symbol, avg(price)
  FROM file('trades.parquet', Parquet)
  GROUP BY symbol
  ORDER BY 2 DESC
  LIMIT 10
"
```

It reads Parquet, CSV, JSON, Avro, ORC, Arrow — and can even write to S3. No server to run.

### 8.4 ClickHouse Cloud

For production, you'll probably just pay ClickHouse Cloud (fully managed, serverless scaling, ~$1/hour for a minimal cluster). Self-hosting at scale requires operations chops — ClickHouse Keeper, backups, merges, monitoring.

---

## 9. Tools for Working with ClickHouse

### 9.1 CLI clients

- **clickhouse-client** — official TCP client. Auto-completion, multi-line editing, progress bars, `--format` for output conversion.
- **clickhouse-local** — SQL over files (see above).
- **chdb** — embedded ClickHouse as a Python/Go/Rust library. No server at all.

### 9.2 GUI clients

- **DBeaver** — universal, solid.
- **DataGrip (JetBrains)** — excellent.
- **TablePlus** — lightweight, macOS-first.
- **Tabix** — open-source web UI (runs in browser, connects over HTTP).
- **clickhouse-ui** and **Play UI** (built into the server at `http://localhost:8123/play`) — handy for quick queries.

### 9.3 Data loading

- **clickhouse-client --query "INSERT ... FORMAT ..."** — the workhorse. Supports 80+ formats (CSV, TSV, JSONEachRow, Parquet, Arrow, Avro, MsgPack, Protobuf, CapnProto...).
- **Kafka engine** — read from Kafka into ClickHouse declaratively.
- **Materialized views** — continuously transform incoming data.
- **ClickPipes** (Cloud) — managed ingestion from Kafka/Postgres/Snowflake.
- **clickhouse-copier** — bulk transfer between clusters.
- **AirByte / Fivetran / Vector / FluentBit / Benthos / Redpanda Connect** — all support ClickHouse as a sink.
- **PeerDB** — streaming replication from Postgres to ClickHouse.

### 9.4 BI and visualization

- **Grafana** — native ClickHouse data source with query editor and $__timeFilter.
- **Metabase** — good for ad-hoc analysis.
- **Superset** — built for column stores, great ClickHouse support.
- **Tableau / PowerBI** — via JDBC/ODBC.

### 9.5 Ops and monitoring

- **clickhouse-backup** (Altinity) — S3-friendly, incremental backups.
- **clickhouse-operator** (Altinity) — run on Kubernetes.
- **Prometheus endpoint** — ClickHouse exposes metrics natively (`/metrics`).
- **system.* tables** — every internal metric is queryable via SQL.

---

## 10. Using ClickHouse from Go

The official Go driver is **`github.com/ClickHouse/clickhouse-go/v2`**. It supports both the fast native protocol and a `database/sql` adapter.

### 10.1 Install

```bash
go get github.com/ClickHouse/clickhouse-go/v2
```

### 10.2 Connect (native protocol)

```go
package main

import (
    "context"
    "log"
    "time"

    "github.com/ClickHouse/clickhouse-go/v2"
    "github.com/ClickHouse/clickhouse-go/v2/lib/driver"
)

func connect() driver.Conn {
    conn, err := clickhouse.Open(&clickhouse.Options{
        Addr: []string{"127.0.0.1:9000"},
        Auth: clickhouse.Auth{
            Database: "default",
            Username: "default",
            Password: "secret",
        },
        Compression: &clickhouse.Compression{
            Method: clickhouse.CompressionLZ4,
        },
        DialTimeout:     time.Second * 5,
        MaxOpenConns:    10,
        MaxIdleConns:    5,
        ConnMaxLifetime: time.Hour,
        Settings: clickhouse.Settings{
            "max_execution_time": 60,
        },
    })
    if err != nil {
        log.Fatal(err)
    }
    if err := conn.Ping(context.Background()); err != nil {
        log.Fatal(err)
    }
    return conn
}
```

### 10.3 Bulk insert — the correct way

ClickHouse is terrible at one-row-at-a-time inserts. **Always batch.** The native driver uses a streaming `Batch` API — you open a batch, append rows in memory, and send them in a single block.

```go
ctx := context.Background()
batch, err := conn.PrepareBatch(ctx, "INSERT INTO trades")
if err != nil {
    log.Fatal(err)
}

for _, t := range trades {
    if err := batch.Append(t.Ts, t.Symbol, t.Price, t.Volume, t.Exchange); err != nil {
        log.Fatal(err)
    }
}
if err := batch.Send(); err != nil {
    log.Fatal(err)
}
```

**Target batch size: 10k–1M rows, or ~10 MB.** Smaller than that and you're leaving throughput on the table.

### 10.4 Async inserts — for when you can't batch

If data arrives one row at a time from many producers, use **async inserts**. ClickHouse server-side buffers rows and flushes them as big batches:

```go
ctx := clickhouse.Context(context.Background(), clickhouse.WithSettings(clickhouse.Settings{
    "async_insert":          1,
    "wait_for_async_insert": 0,   // fire-and-forget
}))

conn.Exec(ctx, "INSERT INTO trades VALUES (?, ?, ?, ?, ?)",
    time.Now(), "BTCUSDT", 50000.1, 100, "BINANCE")
```

### 10.5 Querying

```go
rows, err := conn.Query(ctx, `
    SELECT symbol, avg(price) AS avg_price, count() AS n
    FROM trades
    WHERE ts BETWEEN ? AND ?
    GROUP BY symbol
    ORDER BY n DESC
    LIMIT 10
`, startTime, endTime)
if err != nil {
    log.Fatal(err)
}
defer rows.Close()

for rows.Next() {
    var symbol string
    var avgPrice float64
    var n uint64
    if err := rows.Scan(&symbol, &avgPrice, &n); err != nil {
        log.Fatal(err)
    }
    fmt.Printf("%s: avg=%.2f n=%d\n", symbol, avgPrice, n)
}
```

### 10.6 Struct-based scanning

```go
type Stat struct {
    Symbol   string  `ch:"symbol"`
    AvgPrice float64 `ch:"avg_price"`
    N        uint64  `ch:"n"`
}

var stats []Stat
if err := conn.Select(ctx, &stats, "SELECT ... FROM trades ..."); err != nil {
    log.Fatal(err)
}
```

### 10.7 Streaming large result sets

Don't `Select` a billion rows into a slice. Stream them:

```go
rows, _ := conn.Query(ctx, "SELECT ts, price FROM trades WHERE symbol = ?", "BTCUSDT")
for rows.Next() {
    var ts time.Time
    var price float64
    _ = rows.Scan(&ts, &price)
    // process one row at a time
}
```

### 10.8 HTTP client (alternate)

For serverless / Lambda / cross-language compatibility, use the HTTP interface:

```go
resp, _ := http.Post(
    "http://localhost:8123/?database=default",
    "text/plain",
    strings.NewReader("SELECT count() FROM trades"),
)
```

It's ~2× slower than native but simpler, firewall-friendly, and works with JWT auth.

### 10.9 chdb-go — embedded ClickHouse

```go
import chdb "github.com/chdb-io/chdb-go"

result, _ := chdb.Query("SELECT avg(price) FROM file('trades.parquet')", "JSON")
fmt.Println(result)
```

No server. Single binary. Perfect for CLI tools, data pipelines, one-off analysis.

---

## 11. Where ClickHouse Shines (and Where it Doesn't)

ClickHouse's design — columnar, append-optimized, massively parallel, compression-obsessed — maps to a very specific shape of workload. Below is an exhaustive tour of **where it's a clear win**, **where it's workable but others are better**, and **where it's the wrong tool**.

### 11.1 Clear wins — use ClickHouse

#### 📈 Time-series storage and analytics

The single largest production use case. Metrics, traces, events, tick data, sensor readings — anything timestamped and append-only.

- Ingests millions of rows/sec per node
- Delta/DoubleDelta/Gorilla codecs give 20-100× compression on timestamps + floats
- Built-in `ASOF JOIN`, `RANGE` windows, `toStartOfInterval`, `lag/lead`, forward-fill via `neighbor()`
- TTL for automatic retention + tiered storage (hot NVMe → warm SSD → cold S3)
- Production examples: Uber (M3), CloudFlare, Spotify, Percona PMM, Signoz

**Pitfall:** it's not a "push metrics every second" daemon like Prometheus — you still need a collector (Vector, Fluent Bit, OTel, Kafka) to batch-ingest.

#### 📊 BI & real-time analytics dashboards

The canonical ClickHouse use case. Dashboards that would take minutes on Postgres return in <1s.

- Grafana native plugin with `$__timeFilter` and variables
- Apache Superset, Metabase, Tableau, PowerBI, Redash, Hex, Omni all have mature connectors
- Query cache (v23.5+) makes repeat dashboard loads essentially free
- Projections and materialized views precompute the expensive aggregates
- Handles hundreds of concurrent users with the right `max_concurrent_queries` + settings profiles

**Real-world:** Plausible Analytics, PostHog, GoatCounter, CloudFlare Radar are user-facing dashboards on billions of events, all powered by ClickHouse.

#### 🔍 Observability (logs, metrics, traces)

The fastest-growing ClickHouse category. In 2023-2025 it became the default "cheap alternative to Datadog/Splunk."

- Full-text search via `hasToken`, `match`, bloom filter indexes
- Structured log querying with native JSON (v24.8+) or `Map(String, String)`
- OpenTelemetry exporter support; trace queries over billions of spans in seconds
- 5-10× cheaper than Elasticsearch on storage, 10-100× on aggregation queries
- Purpose-built observability stacks on top: **SigNoz**, **OpenObserve**, **HyperDX**, **Quesma**, **Uptrace**

#### 🧑‍💻 Product analytics

Event-based analytics on user behavior — sessions, funnels, retention, cohorts.

- `windowFunnel` and `sequenceMatch` built-in for funnel analysis
- `retention()` function out of the box
- Sparse primary keys + `LowCardinality` handle tens of thousands of event types trivially
- Used by: PostHog, Mixpanel (partially), Amplitude (internal), June, RudderStack, Jitsu

#### 💰 Fintech / market data / tick analytics

Covered extensively in sections 12 and 16 above — tick stores, TCA, backtesting, surveillance, risk analytics. The combination of `DateTime64(9)`, `ASOF JOIN`, Gorilla compression, and petabyte scale make it first-class here.

#### 📣 Ad-tech / marketing analytics

Impressions, clicks, attribution — trillions of events, real-time reporting to advertisers.

- Used by Criteo, AppsFlyer, Kochava, Mobvista
- `uniqCombined` / HyperLogLog for daily / weekly / monthly unique counters
- Materialized views roll up raw events into reporting cubes

#### 🛰️ IoT and telemetry

Device fleets, vehicles, smart meters — high-cardinality time series.

- Volkswagen, BMW, Tesla, various smart-grid operators
- Handles "millions of devices × events/sec" at single-digit-server economics
- `LowCardinality(String)` on device type / firmware / region + bloom filter on device_id

#### 🛡️ Security analytics, SIEM, audit trails

Log-heavy security workloads — VPN logs, auth logs, DNS, endpoint telemetry.

- Replaces Splunk / QRadar / LogRhythm at 10-100× lower cost
- Bloom filters on suspicious IPs / user agents / file hashes
- Regex functions (`match`, `extractAll`, `multiMatchAny`) for pattern hunting
- Retention of 1-7 years trivially affordable

#### 🧪 ML feature stores and training-data pipelines

ClickHouse makes an excellent **offline feature store** and training-dataset generator.

- SQL over billions of events is the fastest way to build time-aware features
- Windowed aggregates, ASOF joins produce leakage-free point-in-time features
- Integrates with Feast, Hopsworks, and custom feature pipelines
- `chdb` and `clickhouse-connect` plug straight into pandas/Polars/Arrow for ML workflows
- Man Group's ArcticDB pattern: research on Parquet/S3, serve features via ClickHouse

#### 🧬 Vector search (newer, 2024+)

With the `VectorSimilarityIndex` (HNSW) added in 2023-2024, ClickHouse is now viable for hybrid SQL + vector workloads.

- Not as specialized as Pinecone / Qdrant / Milvus for billion-vector pure ANN search
- But **unbeatable** when you want "filter by SQL predicates + vector similarity in one query" — e.g., "show me products similar to X, sold in region Y, in price range Z, in the last 30 days"
- Good fit for recommendation systems where metadata matters

#### 🏞️ Data-lake / lakehouse query engine

ClickHouse reads/writes **Parquet, ORC, Arrow, Iceberg, Delta Lake, Hudi** natively via table functions.

- `s3()`, `iceberg()`, `deltaLake()`, `hudi()` table functions — query in place
- Often competitive with Trino / Spark SQL on typical ad-hoc lakehouse queries, faster on single-table scans
- `INSERT INTO s3(...) SELECT ...` is the simplest ETL you'll ever write
- Pairs well with Iceberg catalogs (Nessie, AWS Glue, Polaris)

#### 📦 Change-data-capture sink (CDC)

With tools like **PeerDB** (ClickHouse-owned since 2024), **Debezium → Kafka → ClickHouse**, or the native **MaterializedPostgreSQL** engine, ClickHouse is now a viable real-time analytics mirror of your operational Postgres/MySQL.

---

### 11.2 Workable, but others may fit better

These are cases where ClickHouse *can* be used, but there's a specialized tool that's usually a better primary pick:

#### Full-text search (primary use case)

ClickHouse has tokenizers, bloom filters, regex, and a (young) inverted index in v24+. It's great for log searches and structured filtering.

- **Better primary tool for relevance-ranked text search:** Elasticsearch, OpenSearch, Meilisearch, Typesense, Vespa
- **Pattern:** if search is 80%+ of your queries → use a search engine. If it's 20% + lots of aggregation → ClickHouse alone works great.

#### Geospatial queries

ClickHouse has basic geo functions (`pointInPolygon`, `greatCircleDistance`, H3 support), but lacks proper spatial indexes.

- **Better for heavy GIS work:** PostGIS (Postgres extension), DuckDB spatial extension, MongoDB geo indexes
- **Pattern:** ClickHouse is fine for simple distance/radius filters; for actual cartographic analysis, use PostGIS.

#### Graph queries / traversals

Not designed for multi-hop graph walks, recursive CTEs work but scale poorly.

- **Better:** Neo4j, Memgraph, TigerGraph, KuzuDB (embedded)

#### Complex OLAP cubes with 20-way joins

ClickHouse's join optimizer improved a lot in 2023-2025 but still isn't Snowflake-level.

- **Better:** Snowflake, BigQuery, Databricks SQL, StarRocks for aggressively join-heavy warehouses
- **Pattern:** denormalize into wide tables for ClickHouse; use a proper DW for heavily normalized star schemas.

#### Ad-hoc notebook analytics on <100 GB

Running a server is overkill for single-user laptop work.

- **Better:** DuckDB (embedded), Polars, chDB (embedded ClickHouse)
- **Pattern:** use DuckDB locally, ClickHouse server when you need multi-user + scale.

#### Pure key-value lookups

Point lookups by arbitrary key are slow — sparse indexes mean reading at least one granule.

- **Better:** Redis, DynamoDB, ScyllaDB, KeyDB, FoundationDB
- **Pattern:** use a KV store for the hot path, ClickHouse for analytics.

#### Queueing / streaming (as the system of record)

ClickHouse has a Kafka engine to *consume* streams, but isn't itself a stream processor or message bus.

- **Better:** Kafka, Redpanda, Pulsar, NATS for pub/sub; Flink, Materialize, RisingWave for streaming SQL

---

### 11.3 When ClickHouse is the WRONG choice

Don't force it:

#### ❌ Transactional workloads (OLTP)

Accounts, orders, inventory, anything needing ACID transactions, row-level locking, referential integrity, or high-concurrency single-row updates.

- **Use instead:** PostgreSQL, MySQL, CockroachDB, YugabyteDB, SQL Server, Oracle
- ClickHouse's mutations are async batch operations. There's no row-level isolation. Don't fight it.

#### ❌ Primary DB for a CRUD app

The entire ClickHouse design assumes append-once, read-many. Every UPDATE is a rewrite of a whole part. Every DELETE is a mutation. You'll hate your life.

- **Use instead:** Postgres/MySQL for the CRUD, ClickHouse for the analytics copy.

#### ❌ Low-volume applications (< ~10M rows total)

Postgres with a good index will outperform ClickHouse on small data and is 10× simpler to run.

- **Rule of thumb:** under ~10M rows / ~1 GB → Postgres. Between 10M-100M → "probably Postgres." Over 100M or multi-TB → ClickHouse starts to dominate.

#### ❌ Hot single-row lookups by arbitrary column

Need to fetch `users.id = 12345` in <1 ms, 10k QPS? That's not ClickHouse's thing.

- **Use instead:** Postgres (indexes), Redis, DynamoDB, any row store with a B-tree.

#### ❌ Relational workloads with heavy foreign-key semantics

ClickHouse doesn't enforce FK constraints and doesn't have a proper optimizer for snowflake schemas with many 1:1/1:N joins.

- **Use instead:** normalized data lives in Postgres; denormalize on the way into ClickHouse.

#### ❌ Full-document storage / schema-less first

The JSON type helps, but if your data is genuinely shape-shifting and you want "schema-on-read + flexible indexing" as a primary pattern…

- **Use instead:** MongoDB, Couchbase, DynamoDB, Elastic (for structured docs)

#### ❌ Geo-distributed multi-region writes with low-latency consistency

ClickHouse can be replicated across regions, but conflict-free geo-write is not its story.

- **Use instead:** Spanner, CockroachDB, YugabyteDB, ScyllaDB

#### ❌ Applications where schema changes hourly

Schema migrations on multi-TB tables can take time. If you're constantly adding/removing/renaming columns, you'll feel friction.

- **Pattern:** for unpredictable schemas, use the JSON type or a `Map(String, String)` column rather than altering DDL constantly.

#### ❌ Regulated workloads requiring row-level security, auditing, fine-grained RBAC

ClickHouse has users, roles, quotas, and row-policy support — but it's less mature than Postgres or SQL Server for strict enterprise compliance.

- **Use instead:** enterprise RDBMS, or run ClickHouse behind an access-control proxy (e.g., chproxy, ClickHouse Cloud RBAC, or a SaaS BI layer).

---

### 11.4 Rule of thumb

```
Is the workload mostly:
├─ Single-row CRUD with transactions?         → Postgres/MySQL
├─ Key-value point lookups?                    → Redis/DynamoDB/Scylla
├─ Full-text search (primary)?                 → Elastic/OpenSearch/Meili
├─ Graph traversal?                            → Neo4j/Memgraph
├─ Geospatial (heavy)?                         → PostGIS
├─ Embedded / notebook-scale?                  → DuckDB / Polars / chDB
├─ Streaming-first (compute on stream)?        → Flink/Materialize/RisingWave
└─ Large-scale aggregation over events/time?   → ClickHouse ✓
```

If your answer to the last one is yes — and particularly if the data is time-shaped, append-only, and grows faster than you can delete it — ClickHouse is almost certainly the right pick.

---

## 12. HFT, Algo Trading, Quant Analytics

This is where ClickHouse earns its reputation. The workload — billions of tick events per day, time-series queries, grouped aggregations, sliding windows — is ClickHouse's sweet spot.

### 12.1 Tick data schema (production-ready)

```sql
CREATE TABLE market.trades (
    ts           DateTime64(9, 'UTC') CODEC(Delta(8), ZSTD(3)),
    exchange     LowCardinality(String),
    symbol       LowCardinality(String),
    price        Float64 CODEC(Gorilla, LZ4),
    qty          Float64 CODEC(Gorilla, LZ4),
    side         Enum8('buy' = 1, 'sell' = -1, 'unknown' = 0),
    trade_id     UInt64,
    seq          UInt64 CODEC(DoubleDelta, LZ4)
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(ts)
ORDER BY (exchange, symbol, ts)
TTL ts + INTERVAL 5 YEAR
SETTINGS index_granularity = 8192;
```

### 12.2 Order book snapshots

```sql
CREATE TABLE market.orderbook_l2 (
    ts       DateTime64(9, 'UTC') CODEC(Delta(8), ZSTD(3)),
    exchange LowCardinality(String),
    symbol   LowCardinality(String),
    bids     Nested(px Float64, qty Float64) CODEC(ZSTD(3)),
    asks     Nested(px Float64, qty Float64) CODEC(ZSTD(3))
)
ENGINE = MergeTree
PARTITION BY toYYYYMMDD(ts)
ORDER BY (exchange, symbol, ts);
```

### 12.3 Real-world queries

**VWAP (volume-weighted average price) per minute:**

```sql
SELECT
    toStartOfMinute(ts) AS minute,
    symbol,
    sum(price * qty) / sum(qty) AS vwap,
    sum(qty)                    AS volume
FROM market.trades
WHERE symbol = 'BTCUSDT'
  AND ts BETWEEN '2026-04-19 00:00:00' AND '2026-04-20 00:00:00'
GROUP BY minute, symbol
ORDER BY minute;
```

**Rolling realized volatility (30s window) using window functions:**

```sql
SELECT
    ts,
    price,
    stddevPop(price) OVER (
        PARTITION BY symbol
        ORDER BY ts
        RANGE BETWEEN 30 PRECEDING AND CURRENT ROW
    ) AS realized_vol_30s
FROM market.trades
WHERE symbol = 'ETHUSDT' AND ts > now() - INTERVAL 1 HOUR;
```

**ASOF JOIN — quote at the time of each trade:**

```sql
SELECT
    t.ts,
    t.symbol,
    t.price AS trade_price,
    q.bid,
    q.ask,
    (q.ask - q.bid) / ((q.ask + q.bid) / 2) AS spread_bps
FROM market.trades AS t
ASOF LEFT JOIN market.quotes AS q
  ON t.symbol = q.symbol AND t.ts >= q.ts
WHERE t.symbol = 'BTCUSDT'
  AND t.ts BETWEEN '2026-04-19' AND '2026-04-20';
```

**ASOF JOIN is a quant-specific superpower.** It joins each left-row with the most recent right-row at or before that timestamp — exactly the operation you need for "what was the prevailing quote when this trade printed?"

### 12.4 Why ClickHouse wins in this space

- **Nanosecond timestamps** — `DateTime64(9)` is first-class
- **ASOF JOIN** — built-in, not a window-function hack
- **Columnar + Gorilla compression** — tick data compresses 20-50×
- **Parallel scans** — a single query scanning a year of trades uses all cores
- **Real-time ingest** — microsecond-fresh data is queryable
- **Low latency** — sub-second P99 on billion-row queries

### 12.5 Actual users

- Crypto exchanges: Binance, Coinbase, Kraken, Deribit all reportedly run ClickHouse for analytics
- Market makers and prop shops: Jump, Jane Street, Virtu have been public about column-store usage (typically kdb+ or internal, but ClickHouse has been taking share)
- Deutsche Börse, Nasdaq use columnar analytics stacks for surveillance
- CloudFlare Radar, PostHog, Plausible, Signoz — all ClickHouse-powered products

---

## 13. Backtester Patterns

This section is a companion to a future article on building a backtester. ClickHouse can serve several distinct roles:

### 13.1 Historical data store

Store every tick for every instrument forever. A year of BTCUSDT trades from a major exchange is ~500M rows (~5-10 GB compressed). You can store decades of data on a single server.

### 13.2 Fast iteration — "what-if" queries

A naive backtester re-reads CSVs for each parameter sweep. With ClickHouse, the data is resident, indexed, and answers complex queries in seconds:

```sql
-- "If my signal fired when 5m returns exceeded 1%, how did the next hour look?"
SELECT
    avg(fwd_return_1h) AS mean_ret,
    stddevPop(fwd_return_1h) AS std_ret,
    count() AS n
FROM (
    SELECT
        ts,
        price,
        (price / lagInFrame(price, 300) OVER (ORDER BY ts) - 1) AS ret_5m,
        (leadInFrame(price, 3600) OVER (ORDER BY ts) / price - 1) AS fwd_return_1h
    FROM market.bars_1s
    WHERE symbol = 'BTCUSDT'
)
WHERE ret_5m > 0.01;
```

One query = one strategy variant. Sweep parameters in a loop.

### 13.3 Event-driven simulation via UDFs

ClickHouse supports **executable UDFs** — you can pipe data through a Python/Rust/Go binary as a SQL function. This lets you run a custom strategy simulator as a SQL function.

```xml
<!-- in /etc/clickhouse-server/functions/strategy.xml -->
<functions>
    <function>
        <type>executable_pool</type>
        <name>my_strategy</name>
        <return_type>Float64</return_type>
        <argument><type>Float64</type></argument>
        <format>TabSeparated</format>
        <command>/opt/strategies/my_strategy.py</command>
    </function>
</functions>
```

```sql
SELECT my_strategy(price) AS pnl FROM market.trades WHERE ...;
```

### 13.4 Pre-aggregated bars (materialized views)

Raw ticks are too fine for most backtests. Build 1s / 1m / 5m / 1h bars as materialized views that update as new trades arrive:

```sql
CREATE MATERIALIZED VIEW market.bars_1m
ENGINE = AggregatingMergeTree
ORDER BY (symbol, minute)
AS SELECT
    symbol,
    toStartOfMinute(ts) AS minute,
    argMinState(price, ts) AS open_state,
    maxState(price)         AS high_state,
    minState(price)         AS low_state,
    argMaxState(price, ts) AS close_state,
    sumState(qty)           AS volume_state
FROM market.trades
GROUP BY symbol, minute;
```

Query:

```sql
SELECT
    minute,
    argMinMerge(open_state)  AS open,
    maxMerge(high_state)     AS high,
    minMerge(low_state)      AS low,
    argMaxMerge(close_state) AS close,
    sumMerge(volume_state)   AS volume
FROM market.bars_1m
WHERE symbol = 'BTCUSDT' AND minute >= now() - INTERVAL 1 DAY
GROUP BY minute ORDER BY minute;
```

Aggregate states are **composable** — you can build 5-minute bars from 1-minute states, 1-hour bars from 5-minute states, etc.

### 13.5 Walk-forward backtests

Use `arrayJoin` and array functions to generate multiple forward windows per signal without self-joins:

```sql
WITH signals AS (
    SELECT ts, price FROM market.trades
    WHERE symbol = 'BTCUSDT' AND ret_5m > 0.01
)
SELECT
    s.ts,
    arrayMap(i -> (i, s.price), [60, 300, 900, 3600]) AS horizons
FROM signals AS s;
```

### 13.6 Practical recommendation

Use ClickHouse for **data preparation, feature engineering, and fast exploratory backtests.** For production event-by-event simulation with complex state, move to a language like Python/Rust/Go — but keep ClickHouse as the tick database and the feature store.

---

## 14. ClickHouse vs The World

### 14.1 vs PostgreSQL (row-store OLTP)

| Aspect | PostgreSQL | ClickHouse |
|---|---|---|
| Storage | Row-based | Column-based |
| Transactions | Full ACID | Basically none |
| Update latency | ms | "eventually" (merges) |
| Aggregation on 1B rows | minutes | sub-second |
| Point lookup by PK | microseconds | slow (~ms+) |
| Use case | Source of truth | Analytics |

### 14.2 vs kdb+/q (quant classic)

kdb+ was the dominant tick database for 25 years. It's extremely fast, but:

- **License**: $100k+/year per core. ClickHouse is free.
- **Language**: q/k is write-only. Half your team can't touch it.
- **Ecosystem**: kdb+ is a silo. ClickHouse plugs into Kafka, S3, Grafana, pandas, and Spark natively.
- **Latency on small queries**: kdb+ wins (~µs). ClickHouse (~ms).
- **Throughput on huge scans**: ClickHouse is competitive or better, especially on distributed clusters.
- **Operationally**: kdb+ requires tribal knowledge. ClickHouse is vastly easier to run.

The trend across the 2020s has been quant teams adopting ClickHouse as a **second tier** (long history, analytics) alongside kdb+ for hot tick data — and often replacing kdb+ entirely in new builds.

### 14.3 vs QuestDB / TimescaleDB / InfluxDB

| DB | Strength | Weakness |
|---|---|---|
| **TimescaleDB** | Postgres ecosystem, full SQL, good ops | Slower on huge scans, doesn't scale horizontally as easily |
| **QuestDB** | Lightning-fast single-node tick DB, low-latency ingestion | Smaller ecosystem, no cluster story, less rich SQL |
| **InfluxDB** | Purpose-built for metrics, nice DSL | Flux is non-standard, 3.x breaks 1.x, limited at petabyte scale |
| **ClickHouse** | Everything above in one package, petabyte scale | Higher ops complexity, overkill for tiny workloads |

For HFT / quant analytics at scale: **ClickHouse and QuestDB are the serious open-source picks**. QuestDB wins on pure latency for single-node. ClickHouse wins on scale, ecosystem, and flexibility.

### 14.4 vs DuckDB

DuckDB is "SQLite for analytics" — an embedded OLAP engine, single-node, file-based. It's *brilliant* for data science on one machine. ClickHouse is its server-side, distributed, multi-user cousin.

- DuckDB: best for notebooks, laptops, one-shot analysis, embedded analytics
- ClickHouse: best for always-on servers, streaming ingest, multi-tenant, clusters

`chdb` closes this gap — it's ClickHouse embedded like DuckDB.

### 14.5 vs Snowflake / BigQuery / Redshift

Those are cloud data warehouses. ClickHouse competes directly with them on price and query speed (often 5-10× cheaper and 2-10× faster on analytical queries) but requires more hands-on work. ClickHouse Cloud narrows the operational gap.

### 14.6 vs Elasticsearch for logs

For structured log analytics, ClickHouse stores the same data in ~5-10× less disk and queries aggregations ~100× faster. Elasticsearch wins on full-text search and ecosystem. Increasingly teams dual-stack: ES for search, ClickHouse for metrics/analytics. See SigNoz, OpenObserve, HyperDX.

### 14.7 vs MongoDB

These two are constantly compared because MongoDB is many teams' "default NoSQL database" — and when that team suddenly needs analytics over a billion documents, they try it in Mongo, get burned, and start looking for alternatives. The honest comparison:

| Dimension | MongoDB | ClickHouse |
|---|---|---|
| Data model | Document store (BSON) | Columnar tables (SQL) |
| Schema | Schema-on-read (flexible) | Schema-on-write (strict), JSON type since v24.8 |
| Primary workload | OLTP / document-centric apps | OLAP / analytics |
| Transactions | Multi-document ACID (since 4.0) | Essentially none |
| Writes per second (single node) | 20k-100k | 1M-5M (batched) |
| Point query by `_id` | <1 ms | Slow (sparse index) |
| Aggregation over 1B docs | Minutes-hours (even with indexes) | Sub-second to seconds |
| Aggregation framework | Pipeline stages (`$group`, `$match`, ...) | Full SQL + window functions |
| Storage footprint (same data) | 3-10× larger | Baseline (aggressive compression) |
| Compression | Snappy / Zstd per block, modest ratios | Delta/Gorilla/ZSTD stacked, extreme ratios |
| Indexing | B-tree, geo, text, hashed, wildcard | Sparse primary + skip indexes (bloom, min/max, set) |
| Full-text search | Built-in (decent) | Tokenizers, bloom, inverted index (newer) |
| Geospatial | Native 2dsphere, great | Basic, not primary strength |
| Clustering | Replica sets + sharding | Sharding + replication via Keeper |
| Operational complexity | Moderate | Moderate-to-high at scale |
| License | SSPL (source-available, non-OSI) | Apache 2.0 |
| Best fit | App backend, content, catalogs, CMS, user profiles | Event logs, metrics, analytics, time-series, tick data |

#### When MongoDB wins

- **Backend for web/mobile apps** — documents, flexible schema, point reads/writes on `_id`
- **Content / catalog / CMS** — heterogeneous nested documents are painful in SQL
- **User profiles, sessions, notifications** — single-document atomicity is perfect
- **Geospatial applications** — Mongo's 2dsphere is one of the best implementations
- **When schema is genuinely unknown and shifting daily**

#### When ClickHouse wins (destroys, really)

- **Any analytics over > ~10M rows/docs** — ClickHouse is usually 50-1000× faster. This is not hyperbole; MongoDB's aggregation pipeline simply wasn't designed to scan huge datasets.
- **Time-series and events** — you will regret picking Mongo for tick data, log ingestion, IoT, or event streams at scale.
- **Reporting dashboards** — Mongo aggregations on large collections routinely take minutes; ClickHouse returns the same in milliseconds.
- **Joins across collections** — Mongo's `$lookup` is notoriously slow; ClickHouse's joins, while not its strongest feature, are far more performant.
- **Storage cost at scale** — Mongo's per-document overhead plus modest compression means 3-10× larger footprint for the same data.

#### The migration pattern (very common)

> "We started on Mongo because it was easy. It got slow around 500M docs. We put ClickHouse next to it, started streaming events via CDC, and moved all reporting + analytics. Mongo is now just the transactional app store."

This is one of the most common stories in ClickHouse adoption — particularly at ad-tech companies, analytics SaaS, IoT platforms, crypto exchanges, and observability products. The two are **complementary**, not interchangeable:

```
┌──────────────┐    CDC (Debezium /    ┌──────────────┐
│  MongoDB     │ ─ change streams) ──> │ Kafka stream │
│  (app data)  │                       └──────┬───────┘
└──────────────┘                              │
      ▲                                       ▼
      │ app reads/writes               ┌──────────────┐
                                       │  ClickHouse  │
                                       │ (analytics)  │
                                       └──────────────┘
```

#### When they overlap — and Mongo still makes sense

- **< 100 GB total**, mixed reads/writes, team already knows Mongo → stay with Mongo
- **Document shapes are wildly heterogeneous** and aggregation needs are modest → Mongo
- **You need multi-document transactions** across arbitrary documents → Mongo

#### When you think you need Mongo but actually want ClickHouse

- "I need flexible schema" — ClickHouse's `JSON` type (v24.8+) now handles this
- "I want to store events" — almost always ClickHouse, not Mongo
- "I want fast aggregations" — ClickHouse, every time
- "I want to query recent + historical data uniformly" — ClickHouse with TTL tiering

#### Performance snapshot (representative, order-of-magnitude)

On a typical ad-tech / analytics workload of ~1B events with ~20 columns:

| Operation | MongoDB | ClickHouse |
|---|---|---|
| Bulk insert 1M rows | ~20-40s | ~1-3s |
| `count()` all rows | ~10-60s | <100 ms |
| `GROUP BY` category + `sum()` | 30s-5min | 200ms-2s |
| Top-10 by aggregate | minutes | <1s |
| Storage on disk | ~200 GB | ~20-40 GB |
| Point query by primary key | <1 ms ✓ | 1-50 ms |

The point-query row is the one cell where Mongo wins — that's literally what it's built for. Everything else is ClickHouse territory.

**Summary:** MongoDB and ClickHouse are not competitors in the traditional sense — they're solving different problems. But they're often evaluated together when a team outgrows MongoDB's analytics ceiling. The mature pattern is to run both: MongoDB as the operational document store, ClickHouse as the analytics engine fed by CDC.

---

## 15. Is ClickHouse Still Relevant in 2026?

Short answer: **more than ever.** The last three years made ClickHouse the de-facto open-source analytics engine, and 2025-2026 cemented it as a serious alternative even to proprietary warehouses.

### 15.1 Signals that matter

- **ClickHouse Inc. (the company) raised a $350M Series C in early 2025** at a $6B valuation, and reportedly $500M+ at a $~6-7B in late 2025 — signaling serious enterprise adoption.
- **GitHub stars**: ~40k+ in 2025, one of the fastest-growing DB repos for the 5th year running.
- **ClickBench** (clickhouse.com/benchmark) — the industry-standard analytical benchmark — shows ClickHouse continuously near the top across 50+ engines tested, and actually wins most small/medium-cluster configurations as of 2025.
- **ClickHouse Cloud** went GA in 2022 and by 2025 runs workloads for Spotify, Anthropic, Lyft, Sony, eBay, Cisco, Tesla, Instacart, and major crypto exchanges.
- **Acquisition of HyperDX (2024)** and **Peerdb (2024)** extended ClickHouse into full observability and streaming Postgres replication.
- **JSON Object type (v24.8, 2024)** finally gave ClickHouse native, schema-on-read JSON — closing a long-standing gap vs. Elastic/Mongo.
- **Parquet & Iceberg first-class support (2024-2025)** — ClickHouse reads/writes Parquet at near-native speed and queries Apache Iceberg tables directly. It now plays inside the modern open lakehouse alongside Spark/Trino/DuckDB.

### 15.2 Where the industry went

The big shift since 2023 has been the **"real-time data warehouse"** category converging on a handful of engines:

- **ClickHouse** — winning the OSS-first crowd
- **Apache Doris / StarRocks** — the Chinese ecosystem's equivalents; strong in APAC
- **Databricks SQL** — backed by Photon (proprietary vectorized engine)
- **Snowflake / BigQuery** — incumbents fighting latency wars
- **DuckDB** — exploding for embedded analytics (different category)

Across this cohort, ClickHouse is the **only one** that is simultaneously:
- Fully open-source (Apache 2.0)
- Production-grade at petabyte scale
- Self-hostable and cloud-hosted
- Competitive on raw query speed with any engine on the market

So yes — in 2026, if you're picking an OSS analytics engine, ClickHouse is still the conservative, safe default. The momentum is clearly with it, not against it.

### 15.3 What changed recently (2024-2026)

Worth knowing if your mental model is pre-2024:

- **Lightweight updates/deletes** (v23.2+) — `DELETE FROM` and `UPDATE` are no longer expensive async mutations; they work like real SQL on small batches.
- **Query cache** (v23.5+) — cache identical query results in RAM, like Redis for your warehouse.
- **Parallel replicas** (v23.3+, stable v24.x) — a single query can now use multiple replicas in parallel. Huge for read-heavy workloads.
- **Vector search** (v23.4+) — `VectorSimilarityIndex`, HNSW, cosine/L2 distance. Competitive with pgvector/Pinecone/Qdrant for medium-scale vector workloads, and you get SQL + analytics on the same data.
- **JSON type** (v24.8+) — real dynamic-schema JSON, not just `String`.
- **BACKUP/RESTORE SQL commands** (mature in 2024) — official, no more tribal rsync scripts.
- **Refreshable Materialized Views** (v23.12+) — scheduled, like BigQuery's scheduled queries.

### 15.4 Headwinds & criticisms

For balance:

- **Operational complexity at scale** remains real. Running a large self-managed cluster requires expertise. Most teams eventually move to ClickHouse Cloud or Altinity.
- **No true UPDATE/DELETE workflow** — despite lightweight mutations, it's still not a transactional DB. If you need frequent row-level changes, Postgres + ClickHouse is the pattern, not ClickHouse alone.
- **Join ergonomics** improved dramatically in 2023-2024 but still lag behind Snowflake/BigQuery for complex analytical joins.
- **SQL dialect** is mostly standard but has ClickHouse-isms. Portability to other engines is imperfect.
- **Competition is fierce**: StarRocks and Apache Doris are genuinely competitive on raw speed, DuckDB owns embedded, and the lakehouse stack (Iceberg + Trino/Spark) is the "safe bet" for pure data-lake workloads.

None of this makes ClickHouse a bad choice — just means "pick the right tool for the job" still applies.

---

## 16. Fintech & Trading Competitors — Detailed Comparison

This is the section you probably came for. Fintech and trading — especially market data, risk analytics, TCA (Transaction Cost Analysis), and backtesting — are the workloads where column-store choice becomes a multi-year commitment. Here's the honest competitive landscape.

### 16.1 The fintech-specific requirements

A capital-markets or trading-analytics data platform typically needs:

| Requirement | Why |
|---|---|
| Nanosecond timestamps | Exchange feeds, PTP-synced clocks, microstructure analysis |
| Time-series semantics | ASOF joins, rolling windows, lag/lead, forward-fill |
| Huge write throughput | 100k-10M+ events/sec for top-of-book + trades |
| Low query latency | Sub-second for dashboards, µs-ms for co-located strategies |
| Efficient compression | Tick archives measure in TBs-PBs per year |
| Rich analytics | Quantiles, VWAP, TWAP, slippage, PnL attribution |
| Python/pandas/Polars interop | Data scientists and quants live in notebooks |
| Regulatory durability | MiFID II, CAT, trade surveillance mandates 5-7+ years retention |

### 16.2 The competitive field in 2026

```
┌─────────────────────────────────────────────────────────────────┐
│                   Fintech tick databases                         │
├───────────────────┬───────────────────┬─────────────────────────┤
│  PROPRIETARY      │  OSS column stores│  SPECIALIZED TS DBs     │
├───────────────────┼───────────────────┼─────────────────────────┤
│  kdb+ / q         │  ClickHouse       │  QuestDB                │
│  OneTick          │  Apache Doris     │  TimescaleDB            │
│  KX Insights      │  StarRocks        │  InfluxDB 3.0           │
│  Refinitiv Tick   │  Apache Pinot     │  Arctic (MongoDB)       │
│  Databricks       │  DuckDB           │  TileDB                 │
│  Snowflake        │                   │  ArcticDB (Man Group)   │
└───────────────────┴───────────────────┴─────────────────────────┘
```

### 16.3 Head-to-head: the serious contenders

Below is a working comparison based on public benchmarks (ClickBench, TSBS — Time-Series Benchmark Suite — and STAC-M3 where available), vendor docs, and field reports through 2025.

> ⚠️ Performance comparisons below are **order-of-magnitude indicators**, not vendor-endorsed numbers. Actual results depend heavily on hardware, schema, and workload. Always benchmark on your data.

#### ClickHouse vs kdb+/q

| Dimension | ClickHouse | kdb+/q |
|---|---|---|
| License cost | Free (OSS) / ~$1-2k/mo for Cloud | $50k-$250k+ per year per core |
| Tick ingest rate (per node) | 1-5M events/sec | 5-20M events/sec |
| Single-query latency (ms, hot cache) | 1-50ms | 0.05-5ms |
| Query latency (cold, 10B rows scan) | 100ms-5s | 50ms-2s |
| Compression ratio (L2 book data) | 10-30× | 5-15× |
| Storage footprint (1y US equities) | ~1-3 TB | ~3-8 TB |
| Cluster/horizontal scale | Excellent | Hard, expensive |
| Developer availability | Huge pool (SQL) | Very small (q is esoteric) |
| Python/pandas interop | First-class (`chdb`, `pyarrow`) | PyKX (ok, but secondary) |
| Strengths | Scale, cost, ecosystem | Ultra-low latency, in-memory tables |

**Verdict:** kdb+ still wins on sub-millisecond latency for small in-memory datasets — it's the gold standard for front-office quant desks. But for everything behind the front office (research, risk, TCA, compliance, backtesting, surveillance), ClickHouse has become the dominant choice because of cost, scale, and the SQL ecosystem. Many firms now run **both**: kdb+ intraday, ClickHouse for the long tail of history and analytics.

#### ClickHouse vs QuestDB

| Dimension | ClickHouse | QuestDB |
|---|---|---|
| Storage model | Columnar MergeTree | Columnar, append-only, mmap |
| Ingest rate (single node) | 1-5M rows/sec | 4-6M rows/sec |
| Query latency (small range) | 1-50ms | 0.1-10ms |
| Query latency (billions of rows) | sub-second with parallel scan | seconds+ (single-node limit) |
| SQL completeness | Very rich (joins, windows, CTEs, ASOF) | Good, somewhat narrower |
| ASOF JOIN | Yes, mature | Yes, original selling point |
| Horizontal scaling | Mature (sharding + replication) | Added 2024 (enterprise), still maturing |
| Clustering | Native (Keeper) | Enterprise feature |
| Ecosystem (BI, Kafka, CDC) | Huge | Growing |
| Python/pandas | `chdb`, `clickhouse-connect` | First-class (parquet dumps) |
| License | Apache 2.0 | Apache 2.0 (OSS) + Enterprise |
| Sweet spot | Petabyte-scale analytics, multi-tenant | Single-node tick ingestion, simpler ops |

**Verdict:** QuestDB is an excellent, purpose-built TSDB — frequently **faster per-node for pure tick ingestion and simple queries**, and its latency is genuinely impressive. ClickHouse wins once your data outgrows a single machine, or when you need rich SQL (joins across many tables, complex subqueries, semi-structured data). Many trading firms start on QuestDB for quick wins, then migrate to ClickHouse when scale demands it.

#### ClickHouse vs TimescaleDB

| Dimension | ClickHouse | TimescaleDB |
|---|---|---|
| Base | Native columnar | Postgres extension |
| Transactions / upserts | Eventual | Full ACID (it's Postgres) |
| Columnar compression | Native, aggressive | Hypertable compression (great but newer) |
| Scan speed (billions of rows) | 5-50× faster | Decent with compression |
| Ingest rate | 1-5M/sec | 200k-1M/sec |
| JOIN ergonomics | Good | Excellent (full Postgres optimizer) |
| Ecosystem | Standalone | Entire Postgres world |
| Best fit (fintech) | Research, backtesting, analytics | Mixed OLTP + time-series (e.g. order tracking + analytics) |

**Verdict:** TimescaleDB is compelling when you already have Postgres and need to add time-series capability without a new stack. For pure-analytics scale, ClickHouse is 5-50× faster on large scans. Use Timescale for **transactional-ish time-series** (live trading state + some analytics), ClickHouse for the **analytics warehouse**.

#### ClickHouse vs InfluxDB 3.0

| Dimension | ClickHouse | InfluxDB 3.0 |
|---|---|---|
| Engine | Native columnar | Apache DataFusion + Parquet |
| Mature since | 2016 | 3.0 launched 2023 |
| Scale in production | Petabytes | TBs (still ramping at petabyte scale) |
| SQL support | Rich SQL | SQL + InfluxQL (Flux deprecated) |
| Cardinality handling | Excellent | Dramatically improved in 3.0 (prior versions struggled) |
| Fintech adoption | Large and growing | Primarily DevOps/IoT; limited fintech presence |

**Verdict:** InfluxDB 3.0 is a great DevOps/IoT metrics DB. It's rarely picked for fintech tick data — the ecosystem and battle-testing isn't there yet.

#### ClickHouse vs StarRocks / Apache Doris

| Dimension | ClickHouse | StarRocks / Doris |
|---|---|---|
| Origin | Yandex (2009) | Baidu/Alibaba forks (2020+) |
| Scan speed | Top-tier | Top-tier (StarRocks often matches or beats CH on ClickBench) |
| Join optimizer | Good (improved a lot) | **Excellent** — cost-based, their main selling point |
| Ingest | Very high | Comparable |
| Ecosystem (west) | Huge | Growing slowly, strong in APAC |
| Fintech adoption | Heavy | Emerging |
| Lakehouse story | Iceberg/Parquet supported | Iceberg/Paimon first-class |

**Verdict:** StarRocks in particular is genuinely competitive on raw query speed and has a better optimizer for complex joins (important in fintech: VaR, position-pnl-risk joins, order-trade stitching). It trails ClickHouse in Western ecosystem and production track record. Worth evaluating if your workload is join-heavy.

#### ClickHouse vs Arctic / ArcticDB (Man Group)

ArcticDB is a **Python-first, Parquet-on-object-storage time-series store**, originally built and open-sourced by Man Group (a major quant fund).

| Dimension | ClickHouse | ArcticDB |
|---|---|---|
| Interface | SQL | Python (pandas DataFrames) |
| Storage | Local / block storage / S3 | S3 / object storage |
| Query engine | Full SQL database | Library + object store reads |
| Latency | Sub-second | Seconds (S3 roundtrips) |
| Best for | Interactive analytics, dashboards | Quant research pipelines, versioned datasets |

**Verdict:** They're complementary, not competitive. ArcticDB is about **reproducible research on versioned Parquet in S3**; ClickHouse is the **query engine for real-time analytics**. Many firms use ArcticDB for researchers and ClickHouse for production analytics.

#### ClickHouse vs OneTick

OneTick is the proprietary tick-analytics database used at many major banks and funds (market surveillance, TCA, backtesting). It's purpose-built for the use case and commands kdb+-class pricing.

| Dimension | ClickHouse | OneTick |
|---|---|---|
| Purpose | General analytics, fintech-capable | Purpose-built for market data |
| Cost | OSS / cheap | Enterprise ($$$) |
| Ingestion from exchanges | DIY (Kafka/ETL) | Managed feed handlers |
| Analytics DSL | SQL | Purpose-built query language |
| Time to market | Days-weeks | Months (but turnkey) |

**Verdict:** OneTick is chosen when you want a **turnkey** market-data platform and have the budget. ClickHouse wins when you have engineering capacity and want cost/flexibility. Several tier-2 banks and fintechs have documented migrations from OneTick to ClickHouse in 2023-2025.

### 16.4 Quick-pick decision table

If your primary concern is: | Pick:
---|---
Ultra-low-latency intraday tick store, cost no object | **kdb+ / OneTick**
Single-node tick DB, simplest ops, fastest per-node ingest | **QuestDB**
Mixed OLTP + time-series on Postgres stack | **TimescaleDB**
Petabyte-scale market history, research, backtest, TCA | **ClickHouse**
Complex-join-heavy risk analytics | **StarRocks** (evaluate vs ClickHouse)
Reproducible, Python-first quant research on S3 | **ArcticDB** (+ ClickHouse for querying)
Embedded analytics in a notebook/service | **DuckDB / chDB**
Open lakehouse with Iceberg, multi-tool access | **Trino / DuckDB / ClickHouse on Iceberg**

### 16.5 A realistic production stack for a modern trading firm (2026)

```
            ┌──────────── Exchange feeds ──────────┐
            │ (FIX, ITCH, OUCH, WebSocket, Kafka)  │
            └──────┬───────────────────────────────┘
                   ▼
        ┌────────────────────┐
        │ Feed handler (Go / │───────┐
        │   C++ / Rust)      │       │
        └────────┬───────────┘       │
                 │                   │
                 ▼                   ▼
    ┌─────────────────────┐   ┌──────────────────┐
    │ kdb+ (intraday hot) │   │ ClickHouse       │◄── streaming via Kafka
    │  strategy / trading │   │ (research,       │     (long-term tick store,
    │  front-office       │   │  backtest, TCA,  │      analytics, dashboards)
    └─────────────────────┘   │  surveillance)   │
                              └──────────────────┘
                                     ▲
                                     │ Python / Polars / Notebooks
                                     │
                              ┌──────┴──────┐
                              │  ArcticDB   │  (versioned feature store
                              │  on S3      │   for ML models)
                              └─────────────┘
```

This "front office on kdb+, middle/back office on ClickHouse" architecture is the most common pattern in the 2024-2026 capital-markets stack. For a greenfield crypto/HFT shop without a kdb+ legacy, the pure ClickHouse (or ClickHouse + QuestDB) stack is overwhelmingly popular.

---

## 17. Best Practices

### 15.1 Ingestion

- **Batch inserts.** Target 10k-1M rows per `INSERT`, or use async inserts. Never do single-row inserts without async mode.
- **Insert into the target table, not through Distributed**, if you can route yourself — it's much faster and more reliable.
- **Match the schema's `ORDER BY` in the insert order** for optimal compression.
- **Use native protocol (port 9000) for bulk loads**, not HTTP.

### 15.2 Schema

- Choose `ORDER BY` based on your actual query filters. Most-common-filter first, time last.
- Wrap low-cardinality strings in `LowCardinality(String)`.
- Use the narrowest numeric type that fits (`UInt32` not `UInt64` for row IDs if you have < 4B rows).
- Avoid `Nullable` unless truly necessary.
- Use explicit `CODEC`s for timestamps and floats — the defaults are conservative.
- Don't over-partition. `PARTITION BY toYYYYMM(ts)` is almost always right.

### 15.3 Queries

- **`PREWHERE`** is often faster than `WHERE` for filters on non-indexed columns. ClickHouse usually picks it automatically, but override if needed.
- **`FINAL`** modifier on `ReplacingMergeTree` / `CollapsingMergeTree` is correct but slow. Use `argMax` patterns instead where possible.
- **Avoid `SELECT *`** — it defeats the columnar advantage.
- **`GROUP BY` on `LowCardinality` columns is very fast.**
- **`LIMIT` with `ORDER BY` uses a special fast path** — use it instead of subqueries when you want top-N.

### 15.4 Joins

- ClickHouse's default `JOIN` loads the right side fully into memory. Put the smaller table on the right.
- For huge joins, use `JOIN ... GLOBAL` (in distributed mode) or pre-aggregate one side.
- Consider `IN (SELECT ...)` instead of `JOIN` when you only need membership — often faster.
- Use `Dictionary` tables for small lookup data. They're cached in RAM and joins become O(1) hash lookups.

### 15.5 Replication & HA

- Always use `ReplicatedMergeTree` in production, even on a single node. You'll thank yourself when you add a replica.
- Use **ClickHouse Keeper** (built-in) rather than ZooKeeper in new deployments.
- Back up with `clickhouse-backup` to S3 regularly.

### 15.6 Resource management

- Tune `max_memory_usage` per query to prevent OOM.
- Use **settings profiles** to isolate workloads (e.g., BI users capped at 8 GB, ETL users uncapped).
- Enable **query log**: `system.query_log` is a free performance monitoring system.

---

## 18. Hidden Gems & Deep Cuts

Things nobody tells you until you've been in the weeds for months.

### 16.1 `EXPLAIN indexes = 1` shows which granules are read

```sql
EXPLAIN indexes = 1
SELECT count() FROM trades WHERE symbol = 'BTCUSDT' AND ts > now() - INTERVAL 1 DAY;
```

Tells you exactly how many granules the primary and skip indexes eliminate. If your query is slow, this reveals whether the index is doing its job.

### 16.2 `SHOW CREATE TABLE ... FORMAT Vertical`

Formats the schema one column per line — much more readable than the default for wide tables.

### 16.3 `-Array`, `-Map`, `-If`, `-OrNull`, `-State`, `-Merge` combinators

Every aggregate function has **combinators** — suffixes that modify behavior:

```sql
sumIf(qty, side = 'buy')          -- conditional sum
countArray(values)                -- count over an array column
avgMap(tag_counts)                -- avg over a map column
quantilesState(0.5, 0.99)(price)  -- intermediate state (for MVs)
uniqExactIf(user_id, is_paid)     -- distinct count with filter
```

Most new users write these as CASE expressions. The combinator form is both faster and shorter.

### 16.4 Sampling

Tables can have a `SAMPLE BY` expression, enabling `SELECT ... SAMPLE 0.1` — scans roughly 10% of rows. Perfect for exploratory queries on huge tables.

```sql
SELECT avg(price) FROM trades SAMPLE 0.01 WHERE symbol = 'BTCUSDT';
```

### 16.5 `argMin` / `argMax`

```sql
SELECT symbol, argMax(price, ts) AS latest_price FROM trades GROUP BY symbol;
```

"Give me the value of X corresponding to the row where Y is max." One function replaces the window-function dance.

### 16.6 `groupArray` + `arrayJoin`

`groupArray` collects values into an array during aggregation. `arrayJoin` expands an array into rows. Combined, they let you do operations that would otherwise require self-joins.

### 16.7 `quantileTDigest` / `quantilesBFloat16`

Approximate quantiles in constant memory. `quantileTDigest` hits ~1% error with tiny state — perfect for SLO dashboards over billions of events.

### 16.8 `HyperLogLog` (`uniq` / `uniqHLL12` / `uniqExact`)

- `uniq` — HyperLogLog, fast, ~1-2% error, default
- `uniqExact` — exact distinct count, needs memory
- `uniqCombined` — hybrid, fast and accurate
- `uniqHLL12` — smaller state

For distinct user/session counting, `uniqCombined` is the sweet spot.

### 16.9 `Materialized` and `Alias` columns

```sql
price     Float64,
log_price Float64 MATERIALIZED log(price),    -- stored, auto-computed
bps_spread Float64 ALIAS (ask - bid) * 10000  -- computed at query time, not stored
```

### 16.10 Projections

A **projection** is a secondary copy of a table with different sort order / aggregation, maintained automatically. Query planner picks the best projection transparently:

```sql
ALTER TABLE trades ADD PROJECTION p_by_exchange (
    SELECT *
    ORDER BY (exchange, ts)
);
```

Now queries filtering on `exchange` fly without a separate table.

### 16.11 `system.asynchronous_metric_log`

Internal metric log sampled every second. You have minute-by-minute system metrics for the life of the server, for free.

### 16.12 `clusterAllReplicas` and `remoteSecure`

Query another cluster or specific replicas from SQL:

```sql
SELECT * FROM clusterAllReplicas('my_cluster', system.parts);
```

Great for fleet-wide introspection.

### 16.13 `file()`, `s3()`, `url()`, `hdfs()` table functions

Query any Parquet/CSV/JSON file in place:

```sql
SELECT count() FROM s3('https://bucket.s3.amazonaws.com/*.parquet', 'KEY', 'SECRET', 'Parquet');
```

### 16.14 `INSERT INTO ... SELECT FROM s3(...)`

One-line ETL. No Spark, no Airflow, no Python.

### 16.15 `MODIFY COLUMN ... MATERIALIZE` backfills

Changing a column's type/codec normally only affects future inserts. Use `ALTER TABLE ... MATERIALIZE COLUMN` to rewrite existing data.

### 16.16 `OPTIMIZE TABLE FINAL DEDUPLICATE BY ...`

Force a merge right now on specific keys. Useful for batch ETL that does "poor man's upserts" via `ReplacingMergeTree`.

### 16.17 `SETTINGS join_algorithm = 'grace_hash'`

For joins that don't fit in memory, this spills to disk. Since v23.x. Saves you from OOM without rewriting the query.

### 16.18 `clickhouse-benchmark`

Official built-in load tester. Point it at any query and get P50/P95/P99/QPS. Beats writing your own.

```bash
clickhouse-benchmark --iterations 1000 \
  --concurrency 16 \
  --query "SELECT count() FROM trades WHERE symbol = 'BTCUSDT'"
```

### 16.19 Window functions with `RANGE` (not just `ROWS`)

```sql
SELECT avg(price) OVER (
    ORDER BY ts
    RANGE BETWEEN INTERVAL 30 SECOND PRECEDING AND CURRENT ROW
) FROM trades;
```

Time-based windows (not row-count-based) without writing custom UDFs.

### 16.20 `EXPLAIN PIPELINE` and `EXPLAIN ESTIMATE`

- `EXPLAIN PIPELINE` — shows the actual parallel execution graph
- `EXPLAIN ESTIMATE` — shows estimated rows/bytes read before running

Most users don't know these exist and stare at `EXPLAIN PLAN` wondering why their query is slow.

---

## 19. Common Pitfalls

- **Single-row INSERTs destroy performance.** Batch them or use `async_insert = 1`.
- **Over-partitioning.** More than ~1000 partitions per table grinds merges to a halt.
- **`Nullable` everything.** Adds 1 byte per value and a second column, and kills some optimizations.
- **Using `String` instead of `LowCardinality(String)`.** Free 5-10× speedup left on the table.
- **`FINAL` in hot paths.** Forces all parts to be merged at query time. Use aggregate patterns instead.
- **Joining two giant tables.** Pre-aggregate one side, or use `Dictionary`.
- **Expecting `DELETE` / `UPDATE` to be instant.** They're async mutations. For high-churn data, use `CollapsingMergeTree` or `ReplacingMergeTree`.
- **Ignoring merge backpressure.** If merges can't keep up with inserts, you hit "too many parts" errors. Monitor `system.parts`.
- **Confusing `ORDER BY` in `SELECT` vs table.** Totally different things.

---

## 20. Operations and Monitoring

The most useful system tables:

| Table | Use |
|---|---|
| `system.query_log` | Every query ever run, with timings |
| `system.parts` | All data parts, sizes, merge status |
| `system.merges` | In-progress merges |
| `system.mutations` | Pending/running `ALTER`s |
| `system.replicas` | Replication lag per table |
| `system.metrics` | Point-in-time counters |
| `system.events` | Cumulative counters since startup |
| `system.asynchronous_metrics` | System health (memory, disk, CPU) |
| `system.settings` | Every tunable with description |
| `system.errors` | Every error recorded by type |
| `system.processes` | Running queries (for `KILL QUERY`) |
| `system.trace_log` | Stack samples (enable with `trace_log`) |

A handful of queries worth saving:

```sql
-- Slowest queries in last hour
SELECT query, query_duration_ms, read_rows, read_bytes, memory_usage
FROM system.query_log
WHERE event_time > now() - INTERVAL 1 HOUR AND type = 'QueryFinish'
ORDER BY query_duration_ms DESC LIMIT 20;

-- Tables by compressed size
SELECT database, table, sum(bytes_on_disk)/1e9 AS gb,
       sum(data_uncompressed_bytes)/1e9 AS uncomp_gb,
       round(uncomp_gb / gb, 2) AS ratio
FROM system.parts WHERE active
GROUP BY database, table ORDER BY gb DESC;

-- Stuck mutations
SELECT * FROM system.mutations WHERE is_done = 0;
```

---

## 21. When NOT to Use ClickHouse

Be honest with yourself. ClickHouse is wrong for:

- Applications needing **row-level ACID transactions** (use Postgres/MySQL)
- Workloads dominated by **single-row updates** (use Postgres/MySQL/Cassandra)
- **Primary key lookups** as the main access pattern (use KV stores / Postgres)
- **Tiny datasets** (<100M rows total) where Postgres or DuckDB is simpler
- **Full-text search** as a primary need (use Elasticsearch / OpenSearch / Meilisearch)
- **Graph traversals** (use Neo4j, Memgraph, or columnar DBs with graph extensions)
- **High-concurrency, low-latency** key-value / row access (ClickHouse shines on scans, not point queries)

A decent rule: ClickHouse is worth adopting once you have either **>100 GB** of data, or **>1000 queries/second** of aggregation load, or **real-time dashboards on event streams**.

---

## 22. Further Reading

- **Official docs**: https://clickhouse.com/docs — genuinely excellent
- **Blog**: https://clickhouse.com/blog — deep technical posts from the core team
- **Altinity Knowledge Base**: https://kb.altinity.com — operations and tuning tips
- **clickhouse-go driver**: https://github.com/ClickHouse/clickhouse-go
- **chdb (embedded)**: https://github.com/chdb-io/chdb
- **Awesome ClickHouse**: https://github.com/ClickHouse/awesome-clickhouse
- **The original paper**: "Column-Stores vs. Row-Stores: How Different Are They Really?" — Abadi et al., 2008 (foundational context for why column stores win on OLAP)

---

## Final Thoughts

ClickHouse is one of those rare pieces of infrastructure that lets a small team do things that used to require a specialized platform. For anything involving **"a lot of events, aggregated over time"** — observability, product analytics, market data, IoT, security logs — it's probably the default choice today.

For quant / HFT / algo trading specifically: if you're not paying for kdb+, you're probably choosing between ClickHouse and QuestDB. Both are excellent. ClickHouse wins on scale, ecosystem breadth, and everything-in-one-box; QuestDB wins on single-node ingestion latency and simplicity. Many teams eventually run both.

The single most important thing to internalize: **ClickHouse is an analytical engine, not a transactional database.** Design around that and it will repay you with performance that feels, frankly, unreasonable.
