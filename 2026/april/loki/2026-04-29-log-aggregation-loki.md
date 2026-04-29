# Log Aggregation with Loki -- The Library Card Catalog of Logs

> The last three days closed two-thirds of the observability triangle. Apr 27 unified instrumentation under OpenTelemetry. Apr 28 went deep on traces -- W3C Trace Context, tail sampling with `loadbalancing`, Tempo's columnar storage, the spanmetrics connector. The marquee click-path of all that work -- "alert fires -> jump to the trace -> click a span -> see the actual log lines from that exact request" -- has one half still unbuilt: the log backend on the receiving end of `traceID`. That backend, in 2026, is **Grafana Loki**, and this doc is the day Loki joins the stack.
>
> The core analogy for the day: **Loki is the Dewey-Decimal sticker on the spine of a book; Elasticsearch is Google for every word in every book.** When you walk into a public library looking for "the chapter on rainforest canopy ecology," you don't ask Google to grep every page of every book in the building. You walk to the right shelf using the spine sticker (Biology -> Botany -> Forest Ecosystems = `577.34`), pull a handful of books, and *flip pages by hand* until you find the chapter. That's the whole bet of Loki: **index only the labels (the spine sticker), store the log content as compressed chunks on cheap object storage (the shelved books themselves), and let queries do the page-flipping at read time.** Elasticsearch took the opposite bet -- build a full-text inverted index of every token in every log line, so any word lookup is a sub-second hash table read. Both work. One is roughly 10x cheaper to operate because the inverted index doesn't exist to maintain. The other has sub-second relevance scoring across years of logs. You pick based on what you actually do with logs, and 80% of teams "filter by service + time range + grep" -- which is exactly the library-shelf workflow.
>
> Two corollaries the library analogy carries. (1) **The labels-as-spine-sticker rule is sacred.** A library that printed a *unique* sticker on every book ("Biology -> Botany -> Forest Ecosystems -> Volume Owned by Patron #4729 -> Last Checked Out at 2026-04-29T13:42") would have a card catalog larger than the library itself. That's what happens when you put `request_id`, `user_id`, `trace_id`, or `pod_name` into Loki labels: the *index* explodes, the chunk files multiply into millions of tiny files on S3, and the ingesters OOM. The 100k-active-streams-per-tenant guideline is the library board telling you "you may not invent a new shelf for every patron." (2) **Structured metadata (Loki 3.0+) is the index card slipped inside the book's front cover.** It carries high-cardinality stuff like `traceID` and `userID` queryably without inflating the spine sticker. Bloom filters in Loki 3.3+ (still experimental in OSS) then make even the index-card lookup fast when enabled. That third tier -- between *spine sticker* (label) and *book content* (log line) -- is what finally unlocks trace-to-logs correlation without the cardinality bomb.

---

**Date**: 2026-04-29
**Topic Area**: loki
**Difficulty**: Intermediate

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| Loki | The Dewey-Decimal-sticker library | Indexes labels only, stores log content as compressed chunks on object storage |
| Stream | One unique combination of labels = one shelf in the library | Identified by `{label1=v1,label2=v2,...}`; the unit of storage and querying |
| Chunk | A bound volume of pages from one shelf, sealed and shipped to the warehouse | Compressed batch of log lines for one stream; flushed from ingester to S3 |
| Label | The spine sticker on a book | Indexed key=value pair that defines which stream a log belongs to; **bounded** values only |
| Structured metadata (3.0+) | An index card slipped inside the book's front cover | Indexed-but-not-stream-defining key-values; the home for `traceID`, `spanID`, `userID` |
| Log line | The actual prose on the pages | Free-form content; searched at query time via line filters |
| Object storage (S3/GCS/Azure/MinIO) | The library's offsite warehouse with infinite cheap shelves | The single source of truth for all chunks and indexes; no Cassandra/ES needed |
| TSDB index (2.10+ default) | The librarian's modern computerized card catalog | Per-tenant index pointing labels -> chunk files; replaces deprecated BoltDB-shipper |
| Bloom filter (3.3+, **experimental**) | "The librarian's quick mental check 'is this in the Botany wing or Romance wing?' before walking over" | Pre-computed probabilistic filter over structured metadata; skips ~70-90% of chunks for needle-in-haystack queries when enabled. Still experimental in OSS Loki -- evaluate before turning on in prod |
| Distributor | The mailroom clerk at the loading dock | Receives writes, validates, fans out to ingesters via consistent hashing, rate-limits per tenant |
| Ingester | The regional warehouse where books wait to be bound | Buffers in-memory chunks per stream, writes WAL, flushes sealed chunks to S3 every 1-2h |
| Querier | The librarian who walks both the shelves and the warehouse | Reads recent data from ingesters AND historical chunks from S3; assembles results |
| Query-frontend | The reference desk that splits a hard request across multiple librarians | Splits queries by time interval, caches results, queues to queriers for parallelism |
| Query-scheduler | The dispatcher pairing queue items to free librarians | Decouples frontends from queriers; lets you scale them independently |
| Compactor | The warehouse manager merging half-empty bins | Consolidates index files daily, applies retention deletes, deduplicates chunks |
| Index-gateway | The card-catalog terminal | Serves TSDB index lookups to queriers without each querier needing local copies |
| Ruler | The librarian who reads a standing alerting query every minute | Evaluates LogQL recording + alerting rules, fires to the same Alertmanager as Prometheus |
| Bloom-planner / -builder / -gateway (3.3+) | Three-person team building and serving the shelf-skip filters | Plans which streams need new blooms, builds them from chunks, serves bloom lookups to queriers |
| Tenant (`X-Scope-OrgID`) | A separate room in the library with its own card catalog | Per-tenant isolation; default is single-tenant `fake` if `auth_enabled: false` |
| Monolithic mode (`-target=all`) | A one-room library | All components in one binary; ~20 GB/day ceiling |
| Simple Scalable Deployment (SSD) | A library with stacks/circulation/admin separated but shared building | Three deployment groups: read/write/backend; ~1 TB/day; the production sweet spot |
| Microservices mode | The Library of Congress with independent buildings per function | Every component a separate Deployment; multi-TB/day; only for hyperscale |
| LogQL | The library catalog query language | Two-mode language: log queries (return lines) and metric queries (PromQL-shaped vectors over logs) |
| Stream selector `{}` | "Walk me to this shelf" | The mandatory first part of every LogQL query: `{app="api", env="prod"}` |
| Line filter (`|=`, `!=`, `|~`, `!~`) | "Hand me the books containing this phrase" | The performance lever -- filters BEFORE parsers is a 5x-100x speedup depending on selectivity |
| Parser (`| json`, `| logfmt`, `| pattern`, `| regexp`) | "Open the book and translate it into structured fields" | Extracts queryable fields from the log line at query time |
| Label filter (`| status >= 500`) | Numeric/string/duration comparison after parsing | Post-parser comparison; only works on extracted fields |
| `| unwrap` | "Pull the numeric value out of this field so I can do arithmetic" | Required step before stat aggregations on log content |
| Grafana Alloy | The new universal forklift -- moves logs, metrics, traces, profiles in one pass | OTel-Collector-based unified agent; absorbed Promtail + Grafana Agent + OTel Collector |
| Promtail | The retired forklift in the corner of the warehouse | EOL March 2, 2026; `alloy convert --source-format=promtail` is the migration path |
| Trace-to-logs pivot | "Click on a flight recording, jump to that pilot's voice transcript" | The killer feature -- click a Tempo span, see Loki logs filtered by trace_id |

---

## The Big Picture -- One Picture of Loki

Before diving in, look at the entire Loki write+read path on one page. Every mention of a component below maps to this diagram:

```
LOKI 3.3+ ARCHITECTURE -- THE LIBRARY ANALOGY
================================================================================

  WRITE PATH (logs flowing in)               READ PATH (queries flowing out)

  +------------+                                +-----------------+
  |  Alloy     |  HTTP /loki/api/v1/push        |  Grafana /      |
  |  (or OTel  |  or /otlp  (OTLP/HTTP clients  |  LogCLI / API   |
  |   Collector|   append /v1/logs themselves)  |                 |
  |   or app   |                                +--------+--------+
  |   OTLP)    |                                         |
  +-----+------+                                         | LogQL
        |                                                v
        v                                       +-----------------+
  +-----------------+                           | QUERY FRONTEND  |
  |   DISTRIBUTOR   |                           | (split by time, |
  | (mailroom clerk)|                           |  cache results) |
  | - validate      |                           +--------+--------+
  | - rate-limit    |                                    |
  | - hash-route    |                                    v
  +--------+--------+                           +-----------------+
           |  per-stream consistent hash        | QUERY SCHEDULER |
           v                                    | (queue + match  |
  +-----------------+      WAL on PVC           |  to free        |
  |    INGESTER     |--------(disk)             |  querier)       |
  |  (regional      |                           +--------+--------+
  |   warehouse)    |                                    |
  | - in-memory     |   ~1-2h later flush                v
  |   chunks/stream |---------+              +----------------------+
  | - WAL on disk   |         |              |       QUERIER        |
  +--------+--------+         |              |  (the librarian)     |
           |                  |              | - reads recent from  |
           | (recent <1-2h    |              |   INGESTER (gRPC)    |
           |  reads)          |              | - reads historical   |
           +-----------------+|              |   from OBJECT STORE  |
                              v              | - asks INDEX GATEWAY |
                  +---------------------+    |   "which chunks?"    |
                  |  OBJECT STORAGE     |<---| - asks BLOOM GATEWAY |
                  |  (S3/GCS/Azure/     |    |   "any chunks I can  |
                  |   MinIO)            |    |   skip?" (3.3+)      |
                  |                     |    +----------+-----------+
                  |  /chunks/...        |               ^
                  |  /index/...  TSDB   |---------------+
                  |  /blooms/... v3 (3.3+)
                  +---------------------+

   BACKGROUND (always running)
   +--------------+   +-------------------+   +----------------+
   |  COMPACTOR   |   | BLOOM PLANNER +   |   |    RULER       |
   | - merge      |   | BLOOM BUILDER     |   | - LogQL alerts |
   |   index files|   | (3.3+, replaces   |   | - LogQL        |
   | - retention  |   |  BLOOM COMPACTOR  |   |   recording    |
   |   deletes    |   |  removed in 3.3)  |   |   rules        |
   | - dedup      |   | - per-tenant      |   | - fires to     |
   |   chunks     |   |   plan + build    |   |   Alertmanager |
   |              |   |   blooms over     |   |   (Apr 23)     |
   | (one only!)  |   |   STRUCTURED      |   |                |
   +--------------+   |   METADATA only)  |   +----------------+
                      +-------------------+

   AT MOST ONE COMPACTOR PER TENANT PER CLUSTER (HA via leader election in 2.7+).
   RUNNING TWO CORRUPTS THE INDEX. THIS IS THE SINGLE MOST DANGEROUS GOTCHA.
```

Six things to remember about this picture:

1. **Object storage is the only durable layer.** Everything else is stateless or has a recoverable WAL. A Loki cluster's "data" is just an S3 bucket; lose every Loki pod tomorrow and you lose at most the last 1-2 hours of un-flushed ingester chunks.
2. **The TSDB index is the modern card catalog.** Since 2.10, BoltDB-shipper is deprecated; you should be on TSDB. The schema migration is real work and is one of the most-skipped upgrades in the wild.
3. **Bloom filters in 3.3+ are *over structured metadata*, not over raw log content.** This is a hard pivot from the 3.0 design. The 3.0/3.2 bloom blocks are incompatible with 3.3+ -- delete and rebuild on upgrade.
4. **Querier reads BOTH ingester (recent) and S3 (historical).** A query for "last 30 minutes" hits ingesters in memory. A query for "last 24 hours" hits both. A query for "last 30 days" mostly hits S3.
5. **Compactor singleton-per-tenant-per-cluster is non-negotiable.** Run two and your index corrupts. Use the built-in leader election (2.7+).
6. **The Ruler plugs into the same Alertmanager as Prometheus (Apr 23).** Alerts on log volume, log error rates, log absence -- same routing tree, same severity matrix, same on-call experience.

---

## Part 1: The Big Bet -- Index the Spine, Not the Pages

Every log aggregation product on earth makes one of two structural bets. Internalize this dichotomy and 80% of Loki's design decisions become obvious:

| Bet | Examples | Storage cost (per GB/month) | Query latency | Best at |
|-----|----------|------------------------------|---------------|---------|
| **Full-text inverted index** | Elasticsearch, OpenSearch, Splunk | $5-10+ (instance-attached SSD + cluster overhead) | 100-500ms for any-token search | "find every log line mentioning `kafka exception` across 90 days" |
| **Label-only index over object storage** | Loki, VictoriaLogs, Quickwit, ClickHouse-as-logs | $0.50-1 (S3 + compressed chunks) | 1-30s typical, depends on time range | "give me all `app=api env=prod` logs in the last hour, then grep `ERROR`" |

These cost numbers move with cluster size, retention, replica count, and whether you're paying for managed services -- treat them as orders-of-magnitude, not budget line items. The cost gap is dominated by two things: **(a) the inverted index itself** (in ES, the index is often *larger* than the raw logs; in Loki, the TSDB index is a few percent of chunk size), and **(b) the storage tier** (ES wants attached SSD with replication factor 2-3; Loki uses S3 at one-third the cost of equivalent block storage and S3's durability removes the need for a Loki-side replica).

The 80/20 rule cuts hard in Loki's favor: **most teams query logs by `service + time + grep`**. That's the exact workflow Loki optimizes for. Teams whose primary use case is "security analytics with 30 days of free-text needle-in-haystack searches" should genuinely look at OpenSearch or Splunk; teams whose primary use case is "incident response, click from an alert to the relevant logs" should look at Loki first.

### What Elasticsearch wins at (be honest)

- **Full-text relevance scoring** at sub-second latency over years of data.
- **Complex aggregations on log content** (faceted search, term aggregations across millions of unique values).
- **Security analytics with rich query DSL** -- the Splunk SPL / KQL / Lucene world has 15 years of practitioner muscle memory.
- **Long retention at huge scale** (>1 year, multi-PB) where the index amortizes over many queries.
- **Sub-second cold queries** -- ES holds index in RAM; Loki goes to S3.

### What Loki wins at

- **Cost** at the operational scale most teams actually run (10-1000 GB/day).
- **Operational simplicity** -- S3 is the storage tier; no cluster sizing dance, no shard rebalancing, no JVM tuning.
- **Native correlation with the Grafana stack** -- trace-to-logs (Tempo), exemplars (Mimir/Prometheus), unified data source experience.
- **OTel-first** -- 3.0 added a native `/otlp` endpoint (clients append `/v1/logs` per OTLP/HTTP spec) that removes the OTel-Collector -> Loki-exporter translation hop.
- **Multi-tenancy with per-tenant retention/limits** built in via `X-Scope-OrgID`.

### The honest 2026 alternatives

- **VictoriaLogs** -- newest contender; smaller community; impressive single-node performance; less mature multi-tenancy.
- **Quickwit** -- inverted-index-on-object-storage; aims to "replace ES on S3" specifically for security/observability.
- **ClickHouse for logs** -- column-store-as-log-store; killer for analytics across all logs but you build the schema yourself.

If your decision is between Loki and ES for *standard observability logging*, Loki is almost always the right answer in 2026. If your use case starts with "we need full-text search across X years," look at the alternatives above before defaulting to ES.

---

## Part 2: The Data Model -- Streams, Chunks, and Why Cardinality Is Everything

The whole product hinges on three concepts: **labels, streams, and chunks**. Internalize this single section and the rest of Loki -- including every gotcha -- falls into place.

### A stream is a unique combination of labels

```
{app="api", env="prod", level="info"}    <- one stream
{app="api", env="prod", level="error"}   <- a DIFFERENT stream
{app="api", env="staging", level="info"} <- a third stream
```

Every stream has its own **family of chunk files** on S3. When the ingester accepts a log line, it:
1. Looks up which stream the line belongs to by hashing its label set.
2. Appends the line to the in-memory chunk for that stream.
3. When the chunk reaches `chunk_target_size` (default 1.5 MiB **uncompressed**; on-disk compressed chunks typically end up 250-500 KB) **or** `max_chunk_age` (default 2h) **or** the stream goes idle for `chunk_idle_period` (default 30m), the chunk is sealed, flushed to S3 under that stream's path, and a new chunk starts.

The key intuition: **a stream is the unit of storage and the unit of querying**. A query like `{app="api"} |= "error"` first asks the index "which streams match `{app="api"}`?", fetches every chunk file for those streams overlapping the time range, decompresses them, and runs the line filter. **The stream selector is the only thing the index helps with.** Everything after the `|=` is brute force over decompressed text.

### The cardinality math (the spice-jar lesson, again)

The number of distinct streams = the cartesian product of the cardinality of each label. From Apr 20 (Prometheus) you already have the muscle memory; the math is identical:

```
{app, env, level}
  app:   10 values    (api, web, worker, ...)
  env:   3 values     (dev, stg, prod)
  level: 5 values     (debug, info, warn, error, fatal)

distinct streams = 10 * 3 * 5 = 150 streams.   FINE.
```

Now the disaster:

```
{app, env, level, pod}
  app:   10
  env:   3
  level: 5
  pod:   500 across the fleet, churning every deploy

distinct streams (steady-state) = 10 * 3 * 5 * 500 = 75,000 streams.   PUSHING IT.
streams created over a 2-week retention window with 10 deploys/day per app =
   10 * 3 * 5 * 500 * 10 deploys * 14 days = 10.5 million streams.    DEAD.
```

That second number is **stream churn**. The active count looks fine but the cumulative index over the retention window OOMs the ingester at WAL replay. Same trap as Prometheus active-series-vs-churn from Apr 20.

The official guideline: **target under 100,000 active streams per tenant.** "Active" means written to within the last `chunk_idle_period`. Beyond that, you're pushing into ingester-OOM territory, S3 file-count explosion (millions of tiny chunks), and query-fan-out latency death.

### The bounded-vs-unbounded label rule

The discipline is identical to Prometheus labels. Label values must be:

| Safe label values | Cardinality killer |
|-------------------|--------------------|
| `app` / `service` (10s) | `request_id` / `trace_id` (millions, unbounded) |
| `env` (3-5) | `user_id` / `account_id` (millions) |
| `level` (5) | `pod_name` (churns every deploy) |
| `cluster` / `region` (10s) | `instance_id` (rotates with autoscaling) |
| `namespace` (100s) | URL path with IDs baked in (`/api/users/12345`) |
| `team` / `tier` (10s) | full SQL query text |
| `host` (bounded by node count) | session ID, IP address of individual client |

**The single most damaging Loki anti-pattern: putting `request_id` / `user_id` / `trace_id` / `pod_name` in labels.** Every one of those values explodes streams, multiplies chunk-file count on S3, and crashes ingesters on WAL replay. Where do they go instead? **Structured metadata** (Part 3) for fields you want queryable, the **log line itself** for fields that are forensic-only.

### The diagnostic command -- run this on every cluster you touch

```bash
logcli series '{}' --analyze-labels --since=1h
```

This dumps a sorted list of which labels have which cardinality across the last hour of ingestion. Instant detection of "someone shoved `pod_name` into a label." Make this the first command you run during any Loki incident, OOM, or onboarding.

A typical healthy output:

```
Total Streams:  4,231
Unique Labels:  9

Label Name      Unique Values   Found In Streams
service         42              4,231
env             3               4,231
level           5               4,231
namespace       18              4,231
host            186             4,231
cluster         3               4,231
...
```

A typical dying-cluster output (pinned to the wall as the warning sign):

```
Total Streams:  843,927               <- !!
Unique Labels:  11

Label Name      Unique Values   Found In Streams
service         42              843,927
env             3               843,927
pod             47,318          843,927   <- THIS
trace_id        892,341         843,927   <- and THIS
...
```

### Active streams vs stream churn

Same lesson as Prometheus active-series-vs-churn. A cluster doing a deploy every 15 minutes with `pod_name` in labels rotates ReplicaSet hashes through stream identity (`api-6f8d9c4b7-abcde` -> `api-7a1e2f3c8-xyzab`), which means every deploy creates a fresh generation of streams and orphans the old ones. The active count looks safe; the index over the retention window OOMs.

Diagnostics:
- `loki_ingester_streams_created_total` -- cumulative streams admitted; take `rate()`.
- `loki_ingester_streams_removed_total` -- the other side of the equation.
- `loki_ingester_memory_streams` -- current live streams in the ingester.

If `rate(loki_ingester_streams_created_total[5m])` is high and roughly matched by `rate(loki_ingester_streams_removed_total[5m])`, you have churn -- an unbounded label is in the mix. Fix by dropping the offending label (`pod_name`, container instance ID, anything that rotates on deploy) and moving it to structured metadata.

### Reserved labels and `__meta_*`

Loki auto-attaches some reserved labels that are stripped before storage but available during relabeling. Same model as Prometheus:

- `__name__` -- not used in Loki the way Prometheus uses it; Loki has no metric names.
- `__path__` -- the file path the log line came from (for the `loki.source.file` Alloy component); commonly relabeled into `filename` or dropped entirely. **Filenames containing UUIDs/timestamps are a stream-explosion source** -- always strip the variable parts.
- `__meta_*` labels -- everything attached by service discovery (Kubernetes pod labels, container names, etc). Use `relabel_rules` to selectively promote the bounded ones into real labels and drop the rest.

The Promtail/Alloy scrape config is the bottleneck for cardinality discipline -- by the time a log line reaches the distributor, the labels are baked. Most cardinality bombs happen because someone copy-pasted a Promtail config that promoted *every* `__meta_kubernetes_pod_label_*` into a real label, and one of those pod labels happened to carry a unique deploy hash.

---

## Part 3: Structured Metadata -- The Index Card Inside the Front Cover (3.0+)

The single biggest data-model addition in Loki 3.0 (April 2024) was **structured metadata**: a third tier between labels and log line, indexed-but-not-stream-defining, that finally gives high-cardinality fields a home.

### The three tiers of "where does this field go?"

| Tier | What goes here | Cardinality budget | Indexed | Defines a stream? |
|------|----------------|---------------------|---------|-------------------|
| **Label** | `service`, `env`, `level`, `namespace`, `cluster`, `host`, `tier` | Bounded (low hundreds per dimension) | YES (TSDB index) | YES |
| **Structured metadata** (3.0+) | `traceID`, `spanID`, `userID`, `requestID`, `pod_name` | High-cardinality OK | YES (per-chunk metadata + bloom filter in 3.3+) | NO |
| **Log line content** | The full log message; everything else | Unlimited | NO (brute-force scan via line filters) | NO |

Before 3.0, this third tier didn't exist. The choice was: bomb the cardinality by labelling `traceID` (catastrophe) or stuff `traceID` into the log line as text and grep it (slow). Structured metadata fixes both.

### How LogQL queries structured metadata

Two access paths:

```logql
# Direct filter on structured metadata field (no parser needed):
{app="api"} | traceID="abc123def456"

# Equivalent to filtering after a JSON parser, but FASTER because:
#  - structured metadata is per-chunk-indexed
#  - 3.3+ bloom filters can skip 70-90% of chunks before download
#
# In 3.3+ this query goes:
#   1. TSDB index: which chunks have streams matching {app="api"}?
#   2. Bloom gateway: of those chunks, which have a structured-metadata
#      bloom that COULD contain traceID="abc123..."?  Skip the rest.
#   3. Querier: download remaining chunks, decompress, filter
```

The bloom-filter optimization in 3.3 is what *can* make "needle in a haystack" trace-id lookups across 30 days of logs go from "I'm getting coffee while the query runs" to "click and wait a few seconds" -- on Grafana Cloud or self-hosted clusters that have opted into the (still-experimental) bloom components. The 3.0/3.2 design built blooms over n-grams of raw log content, which was expensive to build and only helped certain queries; 3.3 pivoted entirely to blooms over structured metadata keys and key-value pairs because that aligns with how OTel logs naturally arrive. As of Loki 3.3-3.5 (April 2026) the bloom subsystem is still labeled experimental in OSS; treat it as opt-in, not as the default query path.

### How structured metadata gets there

Three primary paths:

1. **OTLP ingestion auto-promotes** -- when logs arrive via `/otlp` (3.0+), Loki automatically lifts certain OTLP `LogRecord` fields into structured metadata: `trace_id`, `span_id`, `severity_text`, `severity_number`, plus most non-resource attributes. **This is the cleanest path** -- nothing in your app or Alloy config needs to know about structured metadata.
2. **Alloy `loki.process` stages with `structured_metadata`** -- explicit promotion from extracted fields:

   ```alloy
   loki.process "extract_trace" {
     forward_to = [loki.write.default.receiver]

     stage.json {
       expressions = {
         trace_id = "trace_id",
         span_id = "span_id",
         user_id = "user_id",
       }
     }

     stage.structured_metadata {
       values = {
         trace_id = "",   // empty string = use the extracted field name
         span_id = "",
         user_id = "",
       }
     }
   }
   ```

3. **HTTP push API directly** -- the `/loki/api/v1/push` endpoint accepts structured metadata as a per-line JSON object alongside `[ts, line]`.

### The 3.3 bloom filter incompatibility (gotcha)

If you upgrade from Loki 3.0 or 3.2 to 3.3+, **the existing bloom blocks are incompatible with the new schema (V3) and must be deleted before the new bloom-planner/bloom-builder will run.** The bloom-compactor component was removed entirely. Check your `bloom_*` config sections, delete the existing bloom directory in S3, and let the new components rebuild from scratch over the next few hours. Skipping this leaves your queries slow and your bloom builders crash-looping with cryptic version errors.

---

## Part 4: The Components -- A Tour of the Library Staff

Loki's a microservices system pretending (in monolithic mode) to be a single binary. Every concept below is a `-target=...` flag on the same binary.

### Distributor -- the mailroom clerk

Receives writes via `/loki/api/v1/push` (Loki protocol) or `/otlp` (OTLP, 3.0+; clients append `/v1/logs` per OTLP/HTTP spec):

- **Validates** the request: lineage limits (`per_stream_rate_limit`, `max_line_size` default 256 KB), schema version, label format.
- **Rate-limits** per tenant via `ingestion_rate_mb` (default 4 MB/s **per distributor**, NOT per tenant -- this is the gotcha; multiply by distributor replica count to get per-tenant capacity) and `ingestion_burst_size_mb`.
- **Computes the consistent hash** over the stream's label set to choose `replication_factor` ingesters (default 3).
- **Fans out** the write to all chosen ingesters in parallel; succeeds when `quorum = (RF/2)+1` ack.

A distributor doesn't store anything; it's stateless. Scale them out to absorb write spikes.

### Ingester -- the regional warehouse

Receives writes from distributors over gRPC. Per stream, holds:

- **An in-memory chunk** -- accumulates log lines until full (`chunk_target_size`, default 1.5 MiB **uncompressed**; on-disk compressed chunks are typically 250-500 KB) or aged out (`max_chunk_age`, default 2h) or idle (`chunk_idle_period`, default 30m).
- **A WAL on local disk** (`ingester.wal.enabled: true`, default on in modern Loki) -- every accepted line is fsync'd to disk so a pod crash doesn't lose the in-memory window. Replayed on restart.
- **A flush queue** -- when a chunk seals, ship it to object storage and update the index.

Ingesters are stateful (own a slice of the consistent-hash ring). They run as a StatefulSet with PVCs for the WAL. `replication_factor: 3` means three ingesters hold any given stream, so losing one is graceful.

The **right-sizing knobs** matter operationally:
- `chunk_target_size` too large + low-volume stream = chunks never flush, queries for "yesterday" still hit ingester memory because the data hasn't been sealed and shipped yet. Symptom: 1h+ query latency for "old" data.
- `max_chunk_age` too large = same problem; ingester memory grows unbounded.
- `chunk_idle_period` too short = thousands of tiny chunks per stream on S3, hot-loop the chunk cache.
- WAL disabled = pod restart loses the un-flushed window (could be 30min-2h of data per pod). **Always enable WAL in production.**

### Querier -- the librarian who walks both the shelves and the warehouse

Receives queries from the query-scheduler. For each query:

1. Asks the **index-gateway** "which chunk files match the stream selector + time range?"
2. (3.3+) Asks the **bloom-gateway** "of those chunks, can I skip any based on structured-metadata blooms?"
3. Reads recent data from **ingesters** (gRPC, in-memory) for the un-flushed window.
4. Reads historical data from **object storage** for sealed chunks.
5. Runs the LogQL pipeline (line filters, parsers, label filters, formatters, metric aggregations) over the merged data.
6. Returns results to the frontend.

Queriers are stateless and horizontally scalable. The most common scaling lever in production is "more queriers" -- each query splits into many sub-queries by time range, and the queriers process them in parallel.

### Query-frontend -- the reference desk

The first hop on the read path. Responsibilities:

- **Splits queries by time interval** (`split_queries_by_interval`, default 30m). A query for "last 24h" becomes 48 sub-queries over 30-minute slices, each parallelizable. This is the single biggest read-path scalability lever.
- **Caches results** in memcached or Redis (`results_cache.cache.memcached: ...`). A re-run of the same dashboard query within a cache TTL hits the cache for the slices that haven't changed; only the latest slice is recomputed.
- **Aligns queries to step boundaries** (`align_queries_with_step`) so two dashboard queries sharing a step pull from the same cache slot.
- **Queues sub-queries to the query-scheduler** for fair distribution.

Without a query-frontend, every dashboard refresh re-scans every chunk, every time. Production deployments always run one.

### Query-scheduler -- the dispatcher

Decouples the query-frontend from the queriers. The frontend pushes sub-queries to the scheduler; queriers pull from it when free. Lets you scale frontends and queriers independently. In small deployments you can omit it (the frontend talks directly to queriers); in SSD/microservices it's usually present.

### Compactor -- the warehouse manager

A background process that:

- **Compacts the index** -- merges per-table TSDB index files into larger files for query efficiency.
- **Applies retention** -- deletes chunks older than `limits_config.retention_period` (per-tenant, set in `overrides.yaml`). With `retention_enabled: true` on the compactor, this is what enforces "30 days for default tenant, 90 days for prod-app tenant" rules.
- **Deduplicates chunks** that result from the replication factor (multiple ingesters wrote the same stream's chunk).
- **Per-tenant retention** via `retention_stream` blocks lets you have different retention for different label sets -- "debug logs 7d, prod app logs 90d, audit logs 1y" in the same tenant.

**There can be only one compactor per tenant per cluster.** From 2.7+ the compactor uses leader election on the `compactor_ring` so you can run multiple replicas with one active at a time. **Running two active compactors corrupts the index** -- the most dangerous operational gotcha in all of Loki.

### Index-gateway -- the card-catalog terminal

Serves TSDB index lookups to queriers. Lets you cache the index in a small dedicated tier rather than every querier holding its own copy. Optional in monolithic; standard in SSD/microservices.

### Ruler -- the alerting librarian

Same pattern as Prometheus's ruler (Apr 23): evaluates LogQL recording + alerting rules on a schedule, stores recording-rule outputs as new streams, fires alerts to **the same Alertmanager** you already run for metrics. Two flavors:

- **Local rules** -- rules YAML files in `/etc/loki/rules/<tenant>/`.
- **Remote rules** (`ruler.storage.type: s3`) -- rules stored in object storage; per-tenant; managed via the `loki-mixin-tools` or directly via API.

Loki rule pattern is identical to Prometheus rules:

```yaml
groups:
  - name: api-error-rate
    rules:
      - alert: APIHighErrorRate
        expr: |
          sum by (service) (rate({app="api", level="error"}[5m]))
          /
          sum by (service) (rate({app="api"}[5m]))
          > 0.05
        for: 10m
        labels:
          severity: warning
          team: api
        annotations:
          summary: "API error rate > 5% for 10 minutes"
          runbook_url: "https://runbooks.example.com/api-error-rate"
```

This fires into your existing Alertmanager routing tree (Apr 23) -- `severity: warning` lands in the same Slack channel as your Prometheus warnings; `team: api` routes to the API on-call. **Do not stand up a separate Alertmanager for Loki.** Re-use the production one.

### Bloom-planner / bloom-builder / bloom-gateway (3.3+, experimental)

The 3.3 architecture replaces the removed `bloom-compactor` with three components:

- **bloom-planner** -- per-tenant background task that decides which streams need new bloom blocks built (recently-written streams with structured metadata).
- **bloom-builder** -- pulls plan tasks, downloads the relevant chunks, builds bloom blocks over the structured-metadata key-value pairs, ships to S3.
- **bloom-gateway** -- queriers ask "for this query's selectors and structured-metadata filters, which chunks can I skip?"; the gateway answers via the bloom blocks.

As of 3.3 these components are **still marked experimental** in OSS Loki and "available in public preview for large Grafana Cloud customers." For self-hosted production today (April 2026), running them is a tradeoff -- the speedup on needle-in-haystack queries is real, but the components add operational complexity and have had bugs through 3.3-3.5. Worth deploying if your primary query pattern is `{} | traceID="..."` on multi-day windows; not worth deploying if your queries are mostly time-bounded `{app="..."} |= "ERROR"` (the line filter wins anyway).

---

## Part 5: The Three Deployment Modes -- One Library, Three Buildings

Loki ships as a single binary that behaves differently based on the `-target=` flag. Three canonical configurations:

### Monolithic (`-target=all`) -- the one-room library

Every component runs in one binary, in one pod. Storage still goes to S3, but read+write+compact+rule are all the same process. Simple to operate; sized for small deployments.

```
+-----------------------------+
|  loki -target=all           |
|  (distributor + ingester +  |
|   querier + frontend +      |
|   compactor + ruler + ...)  |
|                             |
|         |        ^          |
+---------+--------+----------+
          |        |
          v        |
        +-----------+
        |    S3     |
        +-----------+
```

**When to use**: <20 GB/day, single-cluster, dev/staging, "I want logging to just work and not be a project."

**Limits**:
- Single-pod write+read concurrency.
- WAL recovery on restart blocks reads.
- Compactor must be a singleton (so you can't HA the monolithic pod without external leader election).

### Simple Scalable Deployment (SSD) -- three rooms in one library

Three target groups, each its own Deployment/StatefulSet:

| Group | `-target=` | Components | Scale |
|-------|------------|------------|-------|
| **Read** | `read` | query-frontend, query-scheduler, querier | Stateless, scale on query load |
| **Write** | `write` | distributor, ingester | StatefulSet (ingester is stateful), scale on ingest rate |
| **Backend** | `backend` | compactor, index-gateway, ruler, bloom-* | Mixed; compactor is singleton via leader election |

```
                     +---------+
                     |  Reads  |  <- Grafana, LogCLI
                     +----+----+
                          |
         +----------------v---------------+
         |   READ pool (query-frontend +  |
         |   queriers, HPA-scaled)        |
         +----------------+---------------+
                          |
         +----------------v-----------------+
         |  WRITE pool (distributor +       |  <- Alloy, OTel
         |  ingester StatefulSet, RF=3)     |
         +----------------+-----------------+
                          |
                          v
                    +-----------+
                    |    S3     |
                    +-----------+
                          ^
         +----------------+-----------------+
         | BACKEND pool (compactor +        |
         | index-gateway + ruler + blooms)  |
         +----------------------------------+
```

**When to use**: 100 GB/day to ~1 TB/day, the production sweet spot for 95% of teams, balanced complexity-vs-scale. The default mode in the official `grafana/loki` Helm chart.

### Microservices (`-target=<each-component>`) -- the Library of Congress

Every component a separate Deployment, scaled independently:

```
distributor (Deployment)        querier (Deployment)
ingester (StatefulSet)          query-frontend (Deployment)
                                query-scheduler (Deployment)
compactor (singleton via LE)    index-gateway (Deployment)
ruler (Deployment)              bloom-planner (Deployment)
                                bloom-builder (Deployment)
                                bloom-gateway (Deployment)
```

**When to use**: >1 TB/day, multi-tenant SaaS, hyperscale. The trade-off: every component is a YAML, every connection is a service, every pod restart is a thing to monitor. Operational tax is real.

### The decision framework

| Volume | Tenants | Mode |
|--------|---------|------|
| <20 GB/day | 1 | **Monolithic** |
| 20 GB - 1 TB/day | 1-10 | **Simple Scalable Deployment** (DEFAULT for production) |
| >1 TB/day OR strict per-tenant isolation | 10+ | **Microservices** |
| Grafana Cloud customer | any | **Use Grafana Cloud Logs** -- this is microservices managed by Grafana Labs |

The single most common mistake: **starting monolithic, hitting the wall around 50 GB/day, then trying to migrate live**. If you have any sense you'll be over 50 GB/day within a year, **start with SSD**. Monolithic-to-SSD migration is mostly a Helm values flip, but it touches every component and requires a maintenance window. SSD-from-day-one is cheap operationally.

---

## Part 6: Storage -- Object Storage Is the Source of Truth

Loki's storage tier is just an object storage bucket. Same bucket, different prefixes:

```
s3://my-loki-bucket/
  chunks/                <- the actual log content, compressed
    fake/                <- tenant ID (default 'fake' if auth_enabled: false)
      <hash>             <- chunk file path is per-stream
  index/
    tsdb/                <- TSDB index files (since 2.10)
      fake/
        index_<time>.tsdb
  blooms/                <- 3.3+ bloom blocks over structured metadata
    fake/
      ...
  rules/                 <- ruler rule files (if ruler.storage.type: s3)
    fake/
      ...
```

### Storage backends

Officially supported: **S3, GCS, Azure Blob Storage, MinIO** (S3-compatible), Swift, **Filesystem** (single-node only). The S3 path is by far the most-tested.

### TSDB index (default since 2.10) -- the modern card catalog

Since Loki 2.10, **TSDB is the recommended index**. It replaced `boltdb-shipper`, which is now deprecated. The TSDB format is a port of Prometheus's TSDB index (yes, the same code), repurposed for log streams.

The schema migration ritual is real work. The `schema_config` block uses **periodic configs with `from:` dates** to migrate forward without rewriting old data:

```yaml
schema_config:
  configs:
    # OLD: BoltDB-shipper, deprecated, do not write new data here
    - from: 2022-01-01
      store: boltdb-shipper
      object_store: s3
      schema: v12
      index:
        prefix: index_
        period: 24h

    # NEW: TSDB, all writes after 2024-04-01 land here
    - from: 2024-04-01
      store: tsdb
      object_store: s3
      schema: v13         # latest schema as of Loki 3.x
      index:
        prefix: index_
        period: 24h
```

The compactor will continue to manage the old BoltDB-shipper data (retention deletes still apply); reads transparently span both schemas based on the time range. The migration is not a one-shot; it's a gradual cutover that takes as long as your retention period (so the BoltDB era ages out naturally).

**Forgetting to add a new periodic config when adopting TSDB is a silent-failure trap**: writes keep going to BoltDB-shipper forever, you never get the TSDB benefits, and one day you upgrade to a Loki version that drops BoltDB-shipper support entirely and your cluster becomes unreadable.

### Tenancy -- single-tenant by default, multi-tenant by header

Loki's tenancy model is dead simple: every request carries an `X-Scope-OrgID` HTTP header, and every component (distributor, ingester, querier, compactor) keys all data by that header.

- **`auth_enabled: false`** (default in many quickstart configs) -- single-tenant; **all data lands in the `fake` tenant**, which is fine for dev but a footgun in production because it's hard to migrate out of.
- **`auth_enabled: true`** + a reverse proxy (oauth2-proxy, NGINX with auth_request, or your existing API gateway) -- the proxy authenticates the client and injects `X-Scope-OrgID: <tenant>` into every request.
- Multi-tenant clients (Alloy, Grafana data source, LogCLI) all support setting the header per-write or per-query.

**Per-tenant overrides** live in `runtime_config` (typically mounted as a ConfigMap):

```yaml
overrides:
  tenant-payments:
    retention_period: 2160h       # 90 days
    ingestion_rate_mb: 32         # higher rate limit
    ingestion_burst_size_mb: 64
    max_query_series: 1000
    max_query_length: 720h        # 30 days max query window
    retention_stream:
      - selector: '{level="debug"}'
        priority: 1
        period: 168h              # debug only kept 7d in this tenant

  tenant-default:
    retention_period: 720h        # 30 days
    ingestion_rate_mb: 4
    ingestion_burst_size_mb: 8
```

**`max_query_series`** and **`max_query_length`** are your protection against "someone wrote `{app=~".+"}` and DoS'd the cluster" -- caps at the per-tenant level, returns a clear error to the client.

---

## Part 7: LogQL Deep Dive -- The Library Catalog Query Language

LogQL is two languages bolted together: **log queries** (return log lines) and **metric queries** (return PromQL-shaped vectors derived from logs). The metric-query side is fundamentally PromQL with a different ingestion source -- everything you learned on Apr 21 transfers verbatim. Log queries are the new material.

### The pipeline structure

Every LogQL query has the shape:

```
{stream selector} | line filter | parser | label filter | formatter | (metric aggregation)
        ^               ^           ^           ^             ^               ^
   index-served    brute-force    extract    post-parse    transform    PromQL-shape
```

The order is **performance-critical**. Filters early, parsers late. A common mistake is parsing first then filtering -- which decompresses every chunk before reducing the data. Get the order wrong and queries are anywhere from 2x slower (low-selectivity filters) to 100x slower (high-selectivity needle-in-haystack); 10x is a fair rule-of-thumb but it's a range, not a constant.

### Stream selectors -- the only thing the index helps with

```logql
{app="api"}                           # exact match
{app="api", env="prod"}               # AND of matchers
{app=~"api|web"}                      # regex (RE2, fully anchored)
{app="api", env!="dev"}               # not equal
{app=~".+", env!~"^test.*"}           # regex with negative regex
```

Same four matcher operators as Prometheus PromQL: `=`, `!=`, `=~`, `!~`. The `=~` and `!~` use RE2 regex and are **fully anchored** (your pattern is wrapped in `^...$`). The biggest gotcha (carried over from Prometheus, Apr 21):

```
{env=~"prod"}      # matches env="prod" ONLY (full anchor)
{env=~".*prod.*"}  # matches env="prod" AND env="staging-prod"
```

A stream selector must match **at least one bounded label** -- `{}` alone is rejected (good; otherwise you'd hot-loop scan every stream). The pragma `{__name__=~".+"}` from Prometheus has no equivalent here; `{job=~".+"}` is the closest moral equivalent.

### Line filters -- the performance lever

Four operators, all O(log size) over the decompressed chunk:

| Operator | Meaning | Example |
|----------|---------|---------|
| `|=` | Contains substring | `{app="api"} |= "ERROR"` |
| `!=` | Does NOT contain substring | `{app="api"} != "healthcheck"` |
| `|~` | Matches RE2 regex | `{app="api"} |~ "ERROR|FATAL"` |
| `!~` | Does NOT match RE2 regex | `{app="api"} !~ "^DEBUG"` |

Line filters are **the single biggest performance lever in LogQL.** Internally:
1. Loki fetches the chunk file.
2. Decompresses each entry.
3. Runs the filter as a substring/regex match on the raw line.
4. **Discards lines that don't match before any further processing.**

If you put a line filter BEFORE a parser, every parser stage runs on a filtered subset (often 1% of the original data). If you put a parser BEFORE the line filter, the parser runs on ALL data. The speedup tracks selectivity: dramatic on rare keywords, modest on common ones.

```logql
# FAST -- filter first, parse only matched lines
{app="api"} |= "ERROR" | json | level="error"

# SLOW -- parse everything, then filter
{app="api"} | json | line=~".*ERROR.*"
```

A subtle 3.x addition: the **`ip()` matcher** for CIDR-based filtering on extracted fields:
```logql
{app="api"} | json | client_ip = ip("10.0.0.0/8")
```
Useful for "give me all requests from internal IPs" without writing a regex.

### Parsers -- translate the log line into structured fields

Five parsers, in the order they're typically reached for:

#### `| json`

Parse the line as JSON. By default, every key becomes an extracted field; nested objects use dotted paths flattened with underscores.

```logql
# Auto-extract all top-level fields
{app="api"} |= "request" | json

# Explicit extraction (faster, no full-tree walk)
{app="api"} |= "request" | json status="status_code", path="request.path"

# After this, you can filter on the extracted fields:
{app="api"} |= "request" | json status="status_code" | status >= 500
```

**Gotcha**: the `json` parser silently produces zero fields when the log line has unescaped control characters, malformed JSON, or non-UTF-8 bytes. Symptom: `| status >= 500` filters out everything because `status` isn't extracted. Diagnose with `| json | __error__ != ""` to surface parse errors.

#### `| logfmt`

Parse `key=value key2="value with spaces"` format. The Grafana stack itself logs in logfmt internally; many Go services do the same.

```logql
{app="api"} | logfmt
{app="api"} | logfmt method, path, status
```

Logfmt is generally faster than JSON parsing because there's no recursive structure -- it's a single linear scan.

#### `| pattern "..."`

Anchored, greedy pattern match for predictable text formats (NGINX access logs, Apache, custom plain-text formats). Use named captures with `<name>`; use `<_>` for "skip this":

```logql
# NGINX access log: 10.0.0.1 - alice [29/Apr/2026:14:00:00 +0000] "GET /api/users HTTP/1.1" 200 1234
{app="nginx"} | pattern `<ip> - <user> [<_>] "<method> <path> <_>" <status> <bytes>`
```

`pattern` is **dramatically faster than `regexp`** for predictable formats because it doesn't backtrack. Reach for `pattern` first; only fall back to `regexp` for genuinely irregular formats.

#### `| regexp "..."`

RE2 regex with named capture groups:

```logql
{app="legacy"} | regexp `(?P<ip>\S+) - (?P<user>\S+) "(?P<request>[^"]+)"`
```

Use sparingly. RE2 is fast as regex engines go, but pattern-matching every line in a 10 GB result set is still expensive.

#### `| unpack`

The inverse of Promtail's old `pack` stage. If you used `pack` to combine multiple labels into one log line for cardinality reduction, `| unpack` reverses it at query time. Mostly relevant if you're carrying old Promtail configs forward.

### Label filters -- numeric/string comparisons after parsing

Once a parser has extracted fields, you can filter on them with type-aware comparisons:

```logql
{app="api"} | json | status >= 500                    # numeric
{app="api"} | json | duration > 250ms                 # duration (with units)
{app="api"} | json | bytes > 1MB                      # bytes (with units)
{app="api"} | json | environment = "production"       # string equality
{app="api"} | json | level =~ "error|warn"            # regex on extracted field
{app="api"} | json | path =~ "^/api/admin.*"          # regex
```

The duration parser understands `ms`, `s`, `m`, `h`; the bytes parser understands `KB`, `MB`, `GB` (decimal) and `KiB`, `MiB`, `GiB` (binary).

### Formatters -- transform the result

Five formatters that reshape the output:

```logql
# line_format -- rewrite the displayed line (Go template)
{app="api"} | json | line_format "{{.method}} {{.path}} -> {{.status}}"

# label_format -- rename or copy extracted fields to labels (for grouping)
{app="api"} | json | label_format service=app

# drop -- remove specific labels from output
{app="api"} | drop level, host

# keep -- keep ONLY these labels (drops the rest)
{app="api"} | keep app, env, level

# decolorize -- strip ANSI color codes from the line
{app="cli-tool"} | decolorize
```

`line_format` is the polish for human-readable Explore views; `label_format` is for renaming a JSON field into a Grafana variable selector.

### Metric queries -- where PromQL knowledge transfers

Wrap a log query in a range aggregation and you get a Prometheus-shaped vector. **Every metric query syntax is identical to PromQL** -- you already know it from Apr 21.

```logql
# Logs as metrics: count error logs per service per minute
sum by (service) (
  rate({app="api", level="error"}[5m])
)

# Latency p99 over 5 minutes from extracted duration field
quantile_over_time(0.99,
  {app="api"} |= "request" | json | unwrap duration [5m]
) by (service)

# Request rate per status code, alerting-ready
sum by (status) (
  rate({app="api"} |= "request" | json | __error__="" [5m])
)

# Error ratio for SLO alerting (logs as metrics)
sum(rate({app="api", level="error"}[5m]))
/
sum(rate({app="api"}[5m]))
```

**The `unwrap` step** is the new piece. To do statistical aggregation (`sum_over_time`, `quantile_over_time`, `avg_over_time`) on a numeric value extracted from a log line, you must `| unwrap <field>` to turn it into a numeric series. Two converter forms handle string-to-numeric coercion: `| unwrap duration(field)` parses Go-style duration strings (`"250ms"` -> `0.25` seconds) and `| unwrap bytes(field)` parses size strings (`"512KB"` -> `524288`). Without these converters, an `unwrap` against a string-typed field silently returns nothing and the metric query produces an empty result with no error -- one of the most confusing LogQL gotchas in production.

```logql
# Total bytes sent per service over 5m
sum by (service) (
  sum_over_time({app="api"} |= "request" | json | unwrap bytes [5m])
)
```

Without `| unwrap`, the only stat aggregations available are counting-based (`rate`, `count_over_time`, `bytes_over_time` for the size of the log lines themselves).

### Range aggregations available

Identical model to Prometheus's range vectors:

| Function | Returns |
|----------|---------|
| `rate({selector}[5m])` | per-second log entry rate |
| `count_over_time({selector}[5m])` | total entries in window |
| `bytes_over_time({selector}[5m])` | total bytes (line size) |
| `sum_over_time(... | unwrap field [5m])` | sum of unwrapped numeric field |
| `avg_over_time(... | unwrap field [5m])` | average |
| `min_over_time(... | unwrap field [5m])` | minimum |
| `max_over_time(... | unwrap field [5m])` | maximum |
| `quantile_over_time(0.95, ... | unwrap field [5m])` | quantile |
| `stdvar_over_time` / `stddev_over_time` | variance / std dev |
| `first_over_time` / `last_over_time` | first/last value |
| `absent_over_time({selector}[5m])` | 1 if NO logs in window |

The `absent_over_time` is the killer for log-based alerting on **silent failures** -- "alert me if the auth service has logged no requests in the last 5 minutes" catches both "service is down" and "load balancer dropped traffic" while a rate-based query would just sit at zero.

### The `$__interval` vs `$__auto_interval` Grafana gotcha

Grafana's `$__interval` template variable interpolates a time range based on dashboard zoom. **In LogQL, the `[range]` selector for metric queries should usually be `$__auto_interval` (or `$__interval`) and the time-range constraint is handled separately.** The exact same gotcha as PromQL (Apr 21): a 1-hour dashboard view + `[5m]` range = correct rate; a 30-day dashboard view + `[5m]` range = many gaps.

Loki has its own quirk on top: **subqueries in LogQL have less mature step alignment than PromQL.** When a Loki dashboard query uses `[1h:5m]` subquery syntax, the alignment artifacts can be surprising. When in doubt, push the aggregation into a **recording rule** (Loki ruler supports them) and dashboard against the materialized series.

### Performance rule of thumb -- pipeline order

Always order the pipeline:

```
{stream selector}        # 1. tightest selector possible
  |= "<keyword>"          # 2. line filter BEFORE parsing -- big speedup (selectivity-dependent)
  | json                  # 3. parse only the surviving lines
  | <field> > 500         # 4. label filter on extracted fields
  | line_format "..."     # 5. cosmetic at the end
```

For metric queries, add the aggregation outermost:

```logql
sum by (service) (              # 6. aggregation, outermost
  rate(
    {app="api"}                  # 1. selector
    |= "ERROR"                   # 2. line filter
    | json                       # 3. parser
    | status = "fatal" [5m]      # 4. label filter
  )
)
```

---

## Part 8: Ingestion Paths -- Who Forklifts Logs Into Loki

Five paths to get logs into Loki. The 2026 answer for greenfield is mostly **Alloy**; everything else has its place.

### Grafana Alloy -- the convergence

In 2024-2025, Grafana Labs unified four agents into one:

- Promtail (log shipper)
- Grafana Agent (Prometheus + telemetry)
- Grafana Agent Operator
- OpenTelemetry Collector (subset)

into a single binary called **Grafana Alloy**, written in Go, configured in **River** (a HCL-like syntax with first-class `--config.format=alloy` support, also supports `--config.format=otelcol` for raw OTel YAML). It is **built on top of OpenTelemetry Collector components** (Apr 27) -- meaning every OTel receiver/processor/exporter is available, plus Grafana-specific blocks for Prometheus and Loki.

The minimal logs-to-Loki Alloy config (Kubernetes DaemonSet, scraping container logs):

```alloy
// 1. Discover container logs from /var/log/pods/*
discovery.kubernetes "pods" {
  role = "pod"
}

// 2. Drop non-running pods, derive labels from K8s metadata
discovery.relabel "pod_logs" {
  targets = discovery.kubernetes.pods.targets

  rule {
    source_labels = ["__meta_kubernetes_pod_phase"]
    regex         = "Running|Succeeded"
    action        = "keep"
  }
  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    target_label  = "namespace"
  }
  rule {
    source_labels = ["__meta_kubernetes_pod_label_app"]
    target_label  = "app"
  }
  rule {
    source_labels = ["__meta_kubernetes_pod_container_name"]
    target_label  = "container"
  }
  // INTENTIONALLY DROPPING pod name -- it's high-cardinality.
  // If you NEED it, put it in structured metadata via loki.process below.
}

// 3. Tail container log files
loki.source.kubernetes "pods" {
  targets    = discovery.relabel.pod_logs.output
  forward_to = [loki.process.parse_json.receiver]
}

// 4. Optional: extract trace_id and promote to structured metadata
loki.process "parse_json" {
  forward_to = [loki.write.default.receiver]

  stage.json {
    expressions = {
      trace_id = "trace_id",
      span_id  = "span_id",
      level    = "level",
    }
  }

  stage.labels {
    values = { level = "" }   // "level" IS bounded (info/warn/error/...) -- safe as label
  }

  stage.structured_metadata {
    values = {
      trace_id = "",          // high-cardinality, NOT a label
      span_id  = "",
    }
  }
}

// 5. Ship to Loki
loki.write "default" {
  endpoint {
    url = "https://loki-gateway.observability.svc/loki/api/v1/push"

    // Multi-tenant: send to a specific tenant
    tenant_id = "tenant-default"
  }

  // Optional: external labels added to every stream
  external_labels = {
    cluster = "prod-us-east",
    region  = "us-east-1",
  }
}
```

A few things this config earns its keep on:

- **`loki.source.kubernetes`** uses the K8s API to read pod logs, no host-path mount of `/var/log/pods` needed. Trade-off: it generates noticeably more kube-apiserver and kubelet load than file-tailing, which matters at high log volume on a DaemonSet -- prefer `loki.source.file` against `/var/log/pods` for prod fleets and reserve this component for environments where the host-path mount isn't available (managed control planes, restricted PSP/PSA policies).
- **The `discovery.relabel` block** is where cardinality discipline lives -- explicitly listed labels are kept, everything else is dropped before stream identity is locked in.
- **The structured-metadata stage** puts `trace_id` in the right tier from the start.
- **External labels** are added by the writer to every stream -- the cluster/region context is automatic without polluting every source's relabel rules.

The same Alloy binary can also handle metrics, traces, and profiles in the same config -- the convergence story from Apr 27 (OTel) extended to logs.

### Promtail (DEAD as of March 2, 2026)

Promtail entered LTS in February 2025 and reached **End-of-Life on March 2, 2026.** As of today (April 29, 2026), it receives no updates, no security patches, and is removed from active distribution. **If you're still running Promtail you need to migrate now.**

The migration tool:
```bash
alloy convert --source-format=promtail --output=alloy.river promtail.yaml
```

This produces an Alloy config that's *behaviorally* equivalent. But there's a non-obvious **silent breakage**:

> **The Prometheus metric names change.**
>
> Promtail emitted metrics like `promtail_files_active_total`, `promtail_read_lines_total`, `promtail_sent_entries_total`. Alloy emits `loki_source_file_files_active`, `loki_source_file_read_lines_total`, `loki_write_sent_entries_total`. **Every alert and dashboard targeting `promtail_*` silently breaks the moment you cut over** -- no errors, just empty queries.

The 2025 field reports recommend a strict order:
1. `grep -r "promtail_" /path/to/alerts /path/to/dashboards` -- find every reference.
2. Map each to its Alloy equivalent ahead of time (the `alloy convert` doc lists them).
3. Stand up Alloy in shadow mode (writes to a different Loki tenant or a test cluster).
4. Verify dashboards render against Alloy metrics.
5. Cut over.

A team that skips step 1 wakes up the next morning to "why is there no alerting on log shipping?" and only finds out after a real incident.

### OpenTelemetry Collector with the `loki` exporter

The path that existed before Loki had native OTLP. The OTel Collector (Apr 27) ships logs to Loki via the `loki` exporter, which translates OTLP `LogRecord` to Loki's push format:

```yaml
exporters:
  loki:
    endpoint: https://loki-gateway.observability.svc/loki/api/v1/push
    headers:
      X-Scope-OrgID: tenant-default
    default_labels_enabled:
      exporter: false
      job: true
      level: true
```

This still works in 2026 but it's the **legacy** path. Two reasons:

1. The translation has lossy behavior on some OTel attributes (resource attributes get flattened into labels in surprising ways).
2. You're running an extra component in the chain when Loki natively speaks OTLP since 3.0.

### Direct OTLP to Loki (3.0+, the modern OTel path)

Loki 3.0 added a native `/otlp` endpoint (the OTLP/HTTP spec appends `/v1/logs` on the client side, so what you configure in clients/exporters is the bare `/otlp` path). Apps using the OTel SDK can ship logs straight to Loki:

```yaml
# In Alloy, this is the otelcol.exporter.otlphttp block:
otelcol.exporter.otlphttp "loki" {
  client {
    endpoint = "https://loki-gateway.observability.svc/otlp"

    headers = {
      "X-Scope-OrgID" = "tenant-default",
    }
  }
}
```

OTLP-ingested logs get **automatic structured-metadata promotion** for `trace_id`, `span_id`, severity, and most non-resource attributes. The resource attributes (`service.name`, `k8s.namespace.name`, `k8s.pod.name`, `service.instance.id`, etc) follow a different rule -- and the default here is the single biggest OTLP-on-Loki gotcha.

**The unsafe default**: out of the box Loki promotes a built-in list of ~17 resource attributes to **labels**, including `k8s.pod.name` and `service.instance.id`. The official docs explicitly call this out as a cardinality risk that exists for backward-compatibility reasons -- not because it's a good idea. Read that twice: a vanilla `helm install loki` with OTLP traffic enabled will, by default, materialize one stream per pod restart. **Always override** the default list:

```yaml
distributor:
  otlp_config:
    # OVERRIDES the unsafe built-in default. Without this block,
    # k8s.pod.name and service.instance.id WILL be promoted to labels.
    default_resource_attributes_as_index_labels:
      - service.name
      - service.namespace
      - deployment.environment
      - k8s.namespace.name
      - k8s.cluster.name
    # k8s.pod.name, service.instance.id, k8s.deployment.name are NOT in
    # this list, so they fall back to structured metadata instead.
```

If you read this section and remember nothing else: the vanilla OTLP default is a cardinality bomb. The override above is mandatory production config, not optional polish.

The decision: **direct OTLP from app SDKs to Loki is fine for low-volume services where you don't need batching/retry/sampling at the collector layer.** For most production deployments, you still want an Alloy or OTel Collector in the path because:

- Batching reduces network round-trips and Loki distributor load.
- Retry/queue handles Loki being briefly unavailable without losing data.
- K8s enrichment (`k8sattributes` processor in OTel, or built-in to Alloy) adds resource attributes the SDK can't see.
- Sampling can be applied centrally (rare for logs, common for traces).

### Fluent Bit with the `loki` output plugin

If your stack is already Fluent Bit (very common at organizations that adopted it before Alloy existed), the **Loki output plugin** is fully supported and well-maintained:

```ini
[OUTPUT]
    name        loki
    match       *
    host        loki-gateway.observability.svc
    port        3100
    tenant_id   tenant-default
    labels      app=$app, env=$env
    label_keys  $level
```

When to choose Fluent Bit over Alloy: **when you already have it deployed and you don't want a second agent**. When to choose Alloy over Fluent Bit: **for greenfield, or when you want one agent to handle metrics/logs/traces/profiles in unified config**.

### Kubernetes events as logs

Cluster-level events (`kubectl get events`) are gold for incident reconstruction. Get them into Loki via Alloy's dedicated component:

```alloy
loki.source.kubernetes_events "cluster_events" {
  forward_to = [loki.write.default.receiver]

  // Optional: only capture events from specific namespaces
  namespaces = ["default", "production", "kube-system"]
}
```

Common gotcha: **events get duplicated across multi-master clusters or HA Alloy deployments** unless you add tenancy-aware deduplication. The simplest mitigation is `replicas: 1` on the Alloy Deployment that runs `loki.source.kubernetes_events` (this component is intrinsically singleton-friendly; the file/journal sources can run on every node).

---

## Part 9: Structured Logging Best Practices

Loki rewards (and penalizes) your logging discipline more than any other backend. Three rules earn their keep:

### 1. JSON or logfmt -- pick one and never look back

Free-form text is a tax forever. Both JSON and logfmt are first-class in LogQL.

- **JSON**: best for nested data, what most modern services emit by default. Pay the parsing cost at query time.
- **logfmt**: flat, faster to parse, what Grafana Labs themselves use for their own services.

Most teams pick JSON. The cost difference at query time is real but rarely the bottleneck (line filters and stream selectors dominate). Consistency across services matters more than the format choice.

### 2. The three-tier field placement rule

Every field in your logs goes in exactly one of three tiers:

| Field example | Tier | Why |
|---------------|------|-----|
| `service`, `env`, `level`, `cluster`, `region`, `namespace` | **Label** | Bounded; defines the stream; queried with `{}` |
| `traceID`, `spanID`, `userID`, `requestID`, `sessionID`, `correlationID`, `pod_name` | **Structured metadata** | High cardinality, queried by exact match (`| traceID="..."`); 3.3+ blooms accelerate this |
| Everything else: error messages, stack traces, business attributes, request bodies, SQL queries | **Log line content** | Searched with `|=`, `!=`, `|~`, `!~`; brute-force scan |

Memorize this table. Every log-cardinality incident in production traces back to a field in the wrong tier.

### 3. Trace ID injection -- the bridge to Apr 28

**Every log line emitted from inside a span must carry the trace_id and span_id.** Without this, the trace-to-logs pivot doesn't work; with it, the whole observability week pays off.

Three implementation paths from the OTel/SDK side:

- **OTel logs SDK (best)** -- the OTel logging SDK auto-attaches `trace_id` and `span_id` from the active span context to every log record. Ship via OTLP and Loki promotes them to structured metadata automatically.
- **Logging-framework bridges** -- Logback's `OpenTelemetryAppender`, Python's `LoggingHandler`, Node's `instrumentation-pino`, .NET's `OpenTelemetry.Logs` -- all auto-inject trace context into log messages before they're written.
- **Manual injection** -- in legacy services, retrieve the current span context and add `"trace_id":"..."` and `"span_id":"..."` to every JSON log line yourself. Painful, error-prone, but works.

In Loki, trace_id ends up in structured metadata (the modern path) or as a parsed JSON field (the older path). The Grafana Tempo data source's `tracesToLogsV2` config knows how to query Loki for `| traceID="..."` to make the click-through work.

### 4. Log levels -- standardized strings

Use a small standardized set: `debug`, `info`, `warn`, `error`, `fatal` (or the OTel-spec `severity_text` values: `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`, `FATAL`). Keep them lowercase, consistent across services.

Loki 3.4+ adds a `detected_level` feature that infers level from common patterns even when the field isn't named `level`. But don't rely on inference -- emit it explicitly. The label discipline kicks in: `level` is bounded (5-6 values), so it's safe as a Loki label.

### 5. Avoid the silent-failure log

A canonical anti-pattern: catch an exception, log a generic `"Error processing request"`, and move on without attaching the exception details, the trace ID, or the input that triggered it. The log line shows up in Loki; the line filter `|= "Error processing"` matches; the operator clicks through and sees... nothing useful.

The fix:

```python
# BAD
try:
    process(req)
except Exception as e:
    log.error("Error processing request")

# GOOD -- structured, rich context, bound to span
try:
    process(req)
except Exception as e:
    log.error(
        "Error processing request",
        extra={
            "error_type": type(e).__name__,
            "error_message": str(e),
            "stack_trace": traceback.format_exc(),
            "request_id": req.id,        # -> structured metadata
            "endpoint": req.endpoint,    # bounded -> safe as label or content
            # trace_id auto-injected by OTel logging SDK
        },
    )
    raise
```

---

## Part 10: The Trace-to-Logs Pivot -- The Marquee Section

This is the payoff of the entire observability week. **An alert fires -> click in Grafana -> see the trace in Tempo -> click a span -> see the Loki log lines from that exact request.**

### The end-to-end picture

```
ALERT FIRING                                       INVESTIGATION CLICKS
======================================================================

Alertmanager      "APIHighLatency on checkout, p99 > 2s, 10m"
       |
       v
Slack/PagerDuty   [Notification]  --click runbook--> Grafana Explore
                                                          |
                                                          | (PromQL panel)
                                                          v
                                                  Latency histogram with
                                                  EXEMPLAR DIAMONDS
                                                  (each diamond = one trace_id)
                                                          |
                                                          | --click exemplar--
                                                          v
                                                  Tempo trace waterfall
                                                  for that trace_id
                                                          |
                                                          | --click span on slow service--
                                                          v
                                                  "View related logs in Loki"
                                                          |
                                                          v
                                                  Loki Explore:
                                                  {service="checkout"}
                                                    | trace_id="abc123def..."
                                                          |
                                                          v
                                                  The actual log lines
                                                  emitted during that request,
                                                  sorted by timestamp,
                                                  including stack traces.
```

### The implementation -- three layers must all be right

Same three-layer model from Apr 27, applied:

**Layer 1: The log record must carry `trace_id`.**
- App-side problem. Either the OTel logs SDK auto-injects it, or a logging bridge does, or you do it manually in code.
- **Most common failure**: traces work via auto-instrumentation, but logs are still written naively to stdout with no trace context. The Loki click-through returns empty because the field isn't there.

**Layer 2: The log record must expose `trace_id` queryably.**
- Schema/ingestion problem. Modern path: ship via OTLP, Loki auto-promotes to structured metadata, Grafana queries `| trace_id="..."`. Older path: emit as JSON top-level field, Grafana extracts with a `derivedField` regex.
- **Critical rule**: never make `trace_id` a Loki *label*. That's the cardinality bomb the whole structured-metadata feature exists to prevent.

**Layer 3: Grafana must know how to construct the LogQL query.**
- Data-source-config problem. The Tempo data source's `tracesToLogsV2` block specifies the linked Loki UID, the tag mapping (`service.name` -> `service_name`), and enables filter-by-trace-id.

### The Grafana Tempo data source config

```yaml
# datasources/tempo.yaml
apiVersion: 1
datasources:
  - name: Tempo
    type: tempo
    uid: tempo
    url: http://tempo-query-frontend.observability.svc:3200
    access: proxy
    jsonData:
      tracesToLogsV2:
        datasourceUid: loki                # <-- THE link
        tags:
          - key: service.name              # OTel resource attribute
            value: service_name            # how it's named in Loki labels
        spanStartTimeShift: '-1h'
        spanEndTimeShift: '1h'
        filterByTraceID: true              # <-- adds | trace_id="..." to query
        filterBySpanID: false              # usually false; trace_id is enough
```

### The reverse direction also works

From a Loki log line containing a trace_id (in structured metadata), you can pivot **back** to the trace via Grafana's "View trace" derived field on the Loki data source:

```yaml
# datasources/loki.yaml
apiVersion: 1
datasources:
  - name: Loki
    type: loki
    uid: loki
    url: http://loki-gateway.observability.svc:3100
    access: proxy
    jsonData:
      derivedFields:
        - name: trace_id
          # Match structured-metadata trace_id; if not in structured metadata,
          # fallback to extracting from JSON message body
          matcherType: label
          matcherRegex: trace_id
          datasourceUid: tempo
          url: '${__value.raw}'
          urlDisplayLabel: 'View trace in Tempo'
```

The combination is a **bidirectional click loop**: alert -> exemplar -> trace -> log -> trace -> log... navigating the same incident from any angle.

### The derived-field gotcha

A common mistake: writing the `derivedFields` regex too greedy. If your regex matches multiple patterns per line, Grafana renders one click-link per match -- a 5 MB log line with many matches produces hundreds of links and **the panel hangs the browser tab**. Always anchor your regex narrowly (`(?:^|\s)trace_id=([a-f0-9]{32})(?:\s|$)`) and test with realistic log lines before deploying.

### Diagnosing a broken click-through -- bisect, don't walk

The chain has six fault domains: (1) app + OTel SDK injecting trace_id, (2) log emission format, (3) collector promoting trace_id as structured metadata (NOT a label), (4) Loki ingestion + correct tenant, (5) Grafana Tempo data source's `tracesToLogsV2` config, (6) the LogQL the click-through actually generates. Walking them top-down is the wrong move at 3 AM. Bisect.

**The single bisecting check**: open Loki Explore directly, paste the trace_id from the Tempo span, set the time range to match the span's start/end, and run a manual LogQL query. The result splits the chain cleanly:

| Result | Interpretation | Where to look next |
|--------|----------------|--------------------|
| Logs return *with* trace_id visible in structured metadata | Layers 1-4 all work | **Layer 5**: Grafana's `tracesToLogsV2` tag mapping, time-window shifts, or wrong `datasourceUid` |
| Logs return but no trace_id field | trace_id never made it into structured metadata | **Layers 1-3**: SDK logger bridge missing, field-name mismatch (`traceId` vs `trace_id`), or collector dropping/relabeling it |
| No logs return at all | Selector or tenant problem | **Layers 4 or 6**: wrong `X-Scope-OrgID`, label drift since the deploy, or LogQL selector matches zero streams |
| Query errors out (parser/syntax error) | LogQL itself is malformed | **Layer 6 only**: structured-metadata filter syntax wrong for the Loki version |

The principle: **rule out the most layers per minute of effort.** Loki Explore sits at the meeting point of "did the data make it" and "is it queryable as expected" -- one query rules out three or four layers in any of the four branches above. Only after the bisect tells you which half of the chain to investigate do you tail Alloy, check the SDK, or open the Tempo data source config.

---

## Part 11: Production Patterns -- The Grown-Up Cluster

A non-exhaustive collection of production patterns from operating Loki at scale.

### Multi-tenancy via `X-Scope-OrgID`

Same model as Prometheus's `tenant` in remote_write. Every component keys data by the `X-Scope-OrgID` HTTP header:

- **Distributor** -- routes writes to per-tenant streams.
- **Ingester** -- separates in-memory state per tenant.
- **Querier** -- only returns data for the requesting tenant.
- **Compactor** -- applies per-tenant retention.
- **Ruler** -- evaluates per-tenant rules.

The auth model is **bring-your-own-proxy**: Loki itself doesn't authenticate; you put oauth2-proxy, NGINX with auth_request, Traefik ForwardAuth, or your existing API gateway in front and have it inject `X-Scope-OrgID` after authenticating.

In Grafana, the data source per tenant looks like:

```yaml
# A separate data source per tenant for clarity
- name: Loki-Production
  type: loki
  uid: loki-prod
  url: http://loki-gateway.observability.svc:3100
  jsonData:
    httpHeaderName1: X-Scope-OrgID
  secureJsonData:
    httpHeaderValue1: tenant-production
```

**Multi-tenant cluster + dashboard with no `X-Scope-OrgID` header = empty results, no error.** This is one of the most-confusing first-day-on-a-new-cluster experiences.

### Per-tenant retention with stream-level overrides

```yaml
# overrides.yaml mounted as a ConfigMap
overrides:
  tenant-production:
    retention_period: 2160h      # 90 days default

    retention_stream:
      # Debug logs: only 7 days
      - selector: '{level="debug"}'
        priority: 1
        period: 168h

      # Audit logs: 1 year
      - selector: '{audit="true"}'
        priority: 1
        period: 8760h

      # Healthcheck noise: 1 day
      - selector: '{path=~".*/health.*"}'
        priority: 2              # higher priority overrides lower
        period: 24h

  tenant-development:
    retention_period: 168h       # 7 days
```

**Per-stream retention is one of Loki's killer cost-control features.** Most teams don't need to keep `level=debug` for 90 days; they just don't want to think about it. The `retention_stream` blocks let you set sane defaults without per-team engineering effort.

### Query-frontend tuning

```yaml
query_range:
  align_queries_with_step: true          # cache hit alignment
  cache_results: true
  results_cache:
    cache:
      memcached:
        addresses: dns+memcached:11211

  parallelise_shardable_queries: true    # split sum-able queries across queriers
  max_retries: 5

frontend:
  log_queries_longer_than: 10s           # log slow queries for tuning
  compress_responses: true
  max_outstanding_per_tenant: 2048

limits_config:
  split_queries_by_interval: 30m         # 30m sub-queries; tune by ingest rate
  max_query_parallelism: 32              # per-tenant concurrent sub-queries
  max_query_series: 500                  # cap streams per query
  max_query_length: 720h                 # 30d max query window
```

### Distributor rate limiting

```yaml
limits_config:
  ingestion_rate_strategy: global         # GLOBAL across distributors (vs per-distributor)
  ingestion_rate_mb: 16                   # MB/sec per tenant
  ingestion_burst_size_mb: 32
  per_stream_rate_limit: 5MB              # MB/sec per stream (catch noisy neighbors)
  per_stream_rate_limit_burst: 15MB
  max_streams_per_user: 100000            # the 100k guideline as a hard cap
  max_line_size: 256KB                    # reject huge log lines
  reject_old_samples: true
  reject_old_samples_max_age: 168h        # don't accept logs older than 7 days
```

The `ingestion_rate_strategy: global` is critical: **the default is `local` which means the rate limit is per-distributor**, so a fleet of 5 distributors gives you 5x the configured limit per tenant. Counter-intuitive, footgun, set to `global` in production.

### Ingester right-sizing

```yaml
ingester:
  lifecycler:
    ring:
      kvstore: { store: memberlist }
      replication_factor: 3                  # 3-way replication of streams
    final_sleep: 0s
    join_after: 60s

  chunk_target_size: 1572864                 # 1.5 MiB uncompressed (default; ~250-500 KB on disk after compression)
  chunk_idle_period: 30m                     # flush after 30m idle
  max_chunk_age: 2h                          # max in-memory time
  chunk_retain_period: 1m                    # keep flushed chunks 1m for ongoing queries
  chunk_block_size: 262144                   # 256KB

  wal:
    enabled: true                            # ALWAYS true in production
    dir: /var/loki/wal
    flush_on_shutdown: true
    replay_memory_ceiling: 4GB               # cap WAL replay memory
```

### Compactor singleton with leader election

```yaml
compactor:
  working_directory: /var/loki/compactor
  shared_store: s3                            # match storage_config.aws

  retention_enabled: true                     # apply per-tenant retention
  retention_delete_delay: 2h                  # safety window before delete
  retention_delete_worker_count: 150

  compactor_ring:
    kvstore:
      store: memberlist                       # leader election via memberlist gossip
```

Run **2-3 replicas** of the compactor Deployment with the ring config above. One will be elected leader; the others stand by. **Never ever run two compactor pods without leader election** -- both will try to compact the same index and corrupt it.

### HA + replication topology

The full picture for an SSD-mode cluster:

```
+--------------------------+  Read pool (HPA on CPU/req-rate)
|  read x N (Deployment)   |  - query-frontend
|                          |  - querier
+--------------------------+  - query-scheduler

+--------------------------+  Write pool (StatefulSet, RF=3)
|  write x 3+ (StatefulSet)|  - distributor
|                          |  - ingester (PVC for WAL)
+--------------------------+

+--------------------------+  Backend pool (mixed)
|  backend x 2-3           |  - compactor (singleton via LE)
|                          |  - index-gateway
|                          |  - ruler
+--------------------------+

+-----------+  +----------+  +----------+
| memcached |  |    S3    |  |   PVC    |
| (results  |  |  bucket  |  |  (WAL)   |
|  cache)   |  +----------+  +----------+
+-----------+
```

---

## Part 12: A Production Helm Snippet

A meaningful YAML showing SSD mode + Alloy DaemonSet writing in. This is not full Terraform (would double the doc length); it's the *load-bearing values* most teams actually edit:

```yaml
# loki-values.yaml -- SSD mode against S3, multi-tenant, structured metadata
# helm repo add grafana https://grafana.github.io/helm-charts
# helm install loki grafana/loki --version 6.x -n observability -f loki-values.yaml

deploymentMode: SimpleScalable

loki:
  auth_enabled: true                         # multi-tenant via X-Scope-OrgID
  schemaConfig:
    configs:
      - from: 2024-04-01
        store: tsdb                          # 2.10+ default
        object_store: s3
        schema: v13                          # latest 3.x schema
        index:
          prefix: index_
          period: 24h

  storage:
    type: s3
    bucketNames:
      chunks: my-loki-chunks
      ruler: my-loki-ruler
      admin: my-loki-admin
    s3:
      region: us-east-1
      # Use IRSA on EKS, no static credentials

  ingester:
    chunk_target_size: 1572864
    max_chunk_age: 2h
    chunk_idle_period: 30m
    wal:
      enabled: true
      dir: /var/loki/wal

  limits_config:
    ingestion_rate_strategy: global          # GLOBAL, not per-distributor
    ingestion_rate_mb: 16
    ingestion_burst_size_mb: 32
    per_stream_rate_limit: 5MB
    max_streams_per_user: 100000             # 100k stream cap
    max_query_series: 500
    max_query_length: 720h                   # 30d max
    split_queries_by_interval: 30m
    retention_period: 720h                   # 30d default

    # OVERRIDE the unsafe default OTLP resource-attribute promotion.
    # Without this block, Loki's built-in default promotes ~17 attributes
    # to labels including k8s.pod.name and service.instance.id (cardinality bomb).
    otlp_config:
      default_resource_attributes_as_index_labels:
        - service.name
        - service.namespace
        - deployment.environment
        - k8s.namespace.name
        - k8s.cluster.name
      # k8s.pod.name, service.instance.id DELIBERATELY EXCLUDED
      # -> they go to structured metadata, not labels.

  compactor:
    retention_enabled: true
    retention_delete_delay: 2h
    compactor_ring:
      kvstore:
        store: memberlist                    # leader election

  ruler:
    enable_alertmanager_v2: true
    alertmanager_url: http://prometheus-kube-prometheus-alertmanager.monitoring.svc:9093
    storage:
      type: s3
      s3:
        region: us-east-1
        bucketnames: my-loki-ruler

  query_range:
    align_queries_with_step: true
    cache_results: true
    results_cache:
      cache:
        memcached_client:
          host: memcached.observability.svc
          service: memcached-client

read:
  replicas: 3
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 12
    targetCPUUtilizationPercentage: 70

write:
  replicas: 3
  persistence:
    enabled: true
    size: 50Gi                                # WAL volume
    storageClass: gp3

backend:
  replicas: 3                                 # 3 backend pods, compactor singleton via LE
  persistence:
    enabled: true
    size: 20Gi

# Per-tenant overrides
runtimeConfig:
  enabled: true
  configMap:
    overrides.yaml: |
      overrides:
        tenant-production:
          retention_period: 2160h             # 90d
          ingestion_rate_mb: 32
          retention_stream:
            - selector: '{level="debug"}'
              priority: 1
              period: 168h                    # debug only 7d
        tenant-default:
          retention_period: 720h              # 30d
```

And a corresponding **Alloy DaemonSet** values snippet:

```yaml
# alloy-logs-values.yaml -- DaemonSet shipping K8s container logs to Loki
# helm install alloy-logs grafana/alloy --version 0.x -n observability -f alloy-logs-values.yaml

controller:
  type: daemonset

alloy:
  configMap:
    create: true
    content: |
      // Discover all running pods
      discovery.kubernetes "pods" {
        role = "pod"
      }

      // Bounded labels only; drop the rest
      discovery.relabel "pod_logs" {
        targets = discovery.kubernetes.pods.targets

        rule {
          source_labels = ["__meta_kubernetes_pod_phase"]
          regex         = "Running|Succeeded"
          action        = "keep"
        }

        // BOUNDED LABELS (safe)
        rule {
          source_labels = ["__meta_kubernetes_namespace"]
          target_label  = "namespace"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_label_app_kubernetes_io_name"]
          target_label  = "app"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_container_name"]
          target_label  = "container"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_label_app_kubernetes_io_component"]
          target_label  = "component"
        }

        // INTENTIONALLY DROPPING:
        //   __meta_kubernetes_pod_name (high cardinality, churns on deploy)
        //   __meta_kubernetes_pod_uid  (cardinality bomb)
        //   any pod_label_* not explicitly promoted above
      }

      // Tail container logs via K8s API
      loki.source.kubernetes "pods" {
        targets    = discovery.relabel.pod_logs.output
        forward_to = [loki.process.parse.receiver]
      }

      // Capture cluster events too
      loki.source.kubernetes_events "events" {
        log_format = "logfmt"
        forward_to = [loki.write.default.receiver]
      }

      // Parse JSON logs and promote trace_id/span_id to structured metadata
      loki.process "parse" {
        forward_to = [loki.write.default.receiver]

        stage.json {
          expressions = {
            level    = "level",
            trace_id = "trace_id",
            span_id  = "span_id",
          }
        }

        // BOUNDED -> safe as label
        stage.labels {
          values = { level = "" }
        }

        // HIGH-CARDINALITY -> structured metadata (not a label)
        stage.structured_metadata {
          values = {
            trace_id = "",
            span_id  = "",
          }
        }
      }

      // Ship to Loki with multi-tenant header
      loki.write "default" {
        endpoint {
          url       = "http://loki-gateway.observability.svc:3100/loki/api/v1/push"
          tenant_id = "tenant-production"
        }

        external_labels = {
          cluster = "prod-us-east-1",
          region  = "us-east-1",
        }
      }

resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }
```

This combination -- Loki SSD + Alloy DaemonSet -- is what 80% of production deployments run in 2026. Every load-bearing decision in the snippets above has a paragraph in this doc explaining why.

---

## Part 13: Production Gotchas (Pinned to the Wall)

In rough order of "how often this bites a new Loki team":

1. **Putting `request_id` / `user_id` / `trace_id` / `pod_name` in labels.** The cardinality killer. Every one of these blows up streams, multiplies chunk-file count on S3, OOMs ingesters at WAL replay. Mitigation: structured metadata or log line content. Diagnostic: `logcli series '{}' --analyze-labels --since=1h`.

2. **Compactor split-brain.** Running 2+ active compactors corrupts the index. Mitigation: `compactor_ring` with leader election (2.7+); run 2-3 replicas, only one is leader at a time. Symptom: weird missing data, query errors about index mismatch, frantic Slack messages from on-call.

3. **`auth_enabled: false` quickstart that gets pushed to prod.** Single-tenant; **all data lands in the `fake` tenant**. Migrating off `fake` later is painful. Mitigation: enable multi-tenant from day one even if you only have one tenant; pick a real tenant name.

4. **Promtail-to-Alloy metric rename breaks every existing alert silently.** `promtail_*` -> `loki_source_file_*`, `loki_process_*`, `loki_write_*`. Mitigation: `grep -r "promtail_"` your alerting rules and dashboards before cutover; map to Alloy equivalents; verify in shadow mode.

5. **BoltDB-shipper deprecation -- writes silently go to the deprecated index.** Forgetting to add a new TSDB `from:` periodic config means your 2026 data lands in BoltDB-shipper format forever, and one day you upgrade past the support cutoff and become unreadable. Mitigation: add the TSDB schema_config block today even on existing clusters; the migration is gradual and safe.

6. **Bloom filter incompatibility between 3.0/3.2 and 3.3+.** The bloom-compactor was removed; new schema (V3) is incompatible with old. Mitigation: delete existing bloom blocks on upgrade; let bloom-planner/builder rebuild over the next few hours.

7. **`chunk_target_size` too large + low-volume stream = chunks never flush.** At 50 lines/hour with 200-byte lines, a 1.5 MiB *uncompressed* target takes ~150 hours to fill -- and queries for "yesterday" still hit ingester memory because the data hasn't been sealed and shipped. Mitigation: `max_chunk_age: 2h` (default) is the safety net; don't raise it; tune `chunk_target_size` down for sparse-stream workloads.

8. **WAL not enabled = pod restart loses 30min-2h of data per pod.** Mitigation: `ingester.wal.enabled: true` (default in modern Loki, but verify); PVCs for the WAL directory; `flush_on_shutdown: true`.

9. **Stream cardinality from `__path__` containing UUIDs/timestamps in filename.** Logs like `/var/log/app/req-<uuid>.log` create one stream per UUID. Mitigation: Alloy `discovery.relabel` rule to either drop the `__path__` label or strip the variable parts before promotion.

10. **Querying `{app=~".+"}` (matching everything) = hot-loop scanning every stream.** A user dropped into Explore types `{` and an autocomplete match accidentally selects `app=~".+"`. Mitigation: `max_query_series` per-tenant cap; `max_query_length` cap; train users on stream selector discipline.

11. **LogQL line filter placed AFTER `| json` parser is dramatically slower.** Plain wrong; the parser runs on every line, the filter runs on the parsed result. The slowdown ranges from 2x (low-selectivity) to 100x (rare-keyword) -- 10x is a fair rule-of-thumb. Mitigation: ALWAYS put `|=` / `!=` / `|~` BEFORE any parser. Code-review your team's recurring LogQL queries.

12. **JSON parser silently produces no fields for malformed input.** Unescaped control characters, non-UTF-8 bytes, malformed JSON -> zero extracted fields, downstream `| status >= 500` filters out everything. Mitigation: surface parse errors with `| __error__ != ""` in diagnostics; sanitize at the source if possible.

13. **Grafana `$__interval` in LogQL `[range]` doesn't behave the same as PromQL.** Loki has its own interval handling; subquery alignment artifacts are more pronounced. Mitigation: prefer recording rules for dashboarded metric queries; `$__auto_interval` is generally safer than `$__interval` for LogQL.

14. **Multi-tenant cluster + dashboard with no `X-Scope-OrgID` header = empty results, no error.** First-day-on-a-cluster confusion. Mitigation: data source per tenant, header in `secureJsonData.httpHeaderValue1`; document the tenant naming scheme.

15. **`ingestion_rate_mb` default is per-distributor, not per-tenant -- surprising under HA.** A fleet of 5 distributors = 5x the configured rate. Mitigation: `ingestion_rate_strategy: global` in `limits_config`.

16. **>100 streams/sec churn from K8s pod rotation in a single namespace = ingester OOM.** A misconfigured Alloy promoting `pod_name` as label + a noisy CronJob namespace = stream churn that kills the ingester. Mitigation: drop `pod_name` from labels; alert on `rate(loki_ingester_streams_created_total[5m])` exceeding baseline.

17. **K8s events ingestion duplicates events without dedup.** Multi-master clusters, multiple Alloy replicas all running `loki.source.kubernetes_events` -> the same event ingested N times. Mitigation: run a single replica of the Alloy Deployment that owns events scraping; use a Deployment (not DaemonSet) for event scraping specifically.

18. **`derived_fields` regex too greedy = Grafana renders MB of links per log line, panel hangs.** Mitigation: anchor your regex narrowly; test with realistic logs; cap the regex to a reasonable portion of the line.

19. **Loki ruler running rules against `{}` matcher = full-cluster scan every interval.** A debug rule someone added for testing in dev gets deployed to prod and DoSes the cluster every minute. Mitigation: code-review all ruler rules before deploy; reject `{}` matchers in CI; cap `max_query_series` at the ruler tenant level too.

20. **Cardinality limit hit silently -- new streams rejected with `429: too many streams`.** Mitigation: alert on `rate(loki_distributor_lines_received_total{reason="rate_limited"}[5m]) > 0`; the alert text should include the diagnostic command (`logcli series ...`).

21. **Direct OTLP from app SDK without a collector = no batching, no retry, no enrichment.** Each app pod opens a direct connection to Loki for every log line. Mitigation: always have an Alloy/OTel Collector in the path for production; reserve direct OTLP for genuinely low-volume side projects.

22. **`__error__` and `__error_details__` labels leak into `by ()` aggregations.** Every parser failure attaches these synthetic labels to the failing line; if you `sum by (service)` over a query that includes a parser, lines with parse errors form their own micro-streams and split your aggregation in surprising ways. Mitigation: drop them early -- `{app="api"} | json | __error__ = ""` filters out parse failures cleanly. Make this the default shape of every parser-using query; only relax it when you're explicitly diagnosing parse errors.

23. **Loki ruler default destination is Loki itself, not Prometheus.** Recording-rule outputs land in the Loki tenant as new streams unless you set `ruler.remote_write.enabled: true` and configure a Prometheus/Mimir endpoint. Many teams discover this only after wondering why their LogQL recording rules don't appear in their Mimir/Prometheus query view. Mitigation: enable `ruler.remote_write` with a `client.url` pointing at Mimir/Prometheus when you want LogQL-derived metrics to live alongside the rest of your metrics.

24. **Log line ordering across streams is not deterministic at the millisecond tie.** Default direction is timestamp DESC; ties at the same millisecond between lines from different streams have no guaranteed order. Bites alert-rule writers ("the alert seems to fire before the log line that matched") and trace-timeline reconstructions. Mitigation: include nanosecond timestamps where you can; for forward-in-time reads, set `direction=forward` (or `--forward` in logcli); don't rely on cross-stream ordering for correctness logic.

---

## Part 14: Decision Frameworks

### 1. Deployment mode

| Volume | Tenants | Pick |
|--------|---------|------|
| <20 GB/day | 1 | **Monolithic** (`-target=all`) |
| 20-1000 GB/day | 1-10 | **Simple Scalable Deployment** (DEFAULT for production) |
| >1 TB/day | 10+ | **Microservices** |
| Don't want to operate it | any | **Grafana Cloud Logs** |

### 2. Where does the field go?

| Cardinality | Query pattern | Tier |
|-------------|---------------|------|
| <100 unique values, bounded | Stream-level filter (`{level="error"}`) | **Label** |
| Thousands+ unique, queried by exact match | Direct lookup (`| traceID="..."`) | **Structured metadata** (3.0+) |
| Unbounded, searched as substring | Grep (`|= "out of memory"`) | **Log line content** |

### 3. Collector

| Situation | Pick |
|-----------|------|
| Greenfield K8s logging | **Alloy DaemonSet** (the 2026 default) |
| Already deployed Fluent Bit cluster-wide | **Fluent Bit** with `loki` output (don't add a second agent) |
| Heavy OTel investment, want one collector for all signals | **OTel Collector** with `loki` exporter or direct OTLP |
| Low-volume side project, single service | **Direct OTLP from app SDK to Loki** (3.0+ native endpoint) |
| Migrating from Promtail | **Alloy** via `alloy convert` -- Promtail is EOL March 2026 |

### 4. Loki vs ELK/EFK vs alternatives

| Use case | Pick |
|----------|------|
| Standard observability logging, K8s, cost-conscious | **Loki** |
| Full-text security analytics over years of logs | **Elasticsearch** / **OpenSearch** |
| Splunk-shop with deep SPL muscle memory | **Splunk** (don't migrate just for cost) |
| Want ES-killer on object storage | **Quickwit** (rising 2024-2026) |
| Want column-store for arbitrary log analytics | **ClickHouse** with a logs schema |
| Lightweight single-node alternative | **VictoriaLogs** |
| Already on Datadog Logs / New Relic / Honeycomb Logs | **Stay** -- migrating logs is a slog rarely worth the cost |

### 5. Loki ruler vs Prometheus ruler for alerts

| Signal | Pick |
|--------|------|
| Metric-based SLO burn rate (request rate, error rate, latency from histograms) | **Prometheus ruler** -- this is what it's for |
| "No logs in last 5 minutes" silent-failure alerts | **Loki ruler** with `absent_over_time({job="api"}[5m])` |
| Log volume thresholds ("too many ERROR logs") | **Loki ruler** |
| Pattern-based alerts ("OOMKilled appeared in any pod") | **Loki ruler** with line filter |
| Metrics derived from logs (counting log levels per service) | **Either**; recording rule in Loki cheaper than re-deriving in Prometheus |

> **Note on Loki recording rules**: by default Loki ruler writes recording-rule outputs back into Loki itself (as new streams in the same tenant), not into Prometheus. If you want them queryable alongside the rest of your metrics in Mimir/Prometheus, set `ruler.remote_write.enabled: true` and point `ruler.remote_write.client.url` at the Mimir/Prometheus remote-write endpoint. Without that, your "Loki-derived metrics" are invisible to PromQL panels in Grafana even though they exist.

---

## Part 15: A LogQL Cheat Sheet

```logql
# === LOG QUERIES ===

# Stream selector + line filter (the 80% query)
{app="api", env="prod"} |= "ERROR"

# Multiple line filters (chained)
{app="api"} |= "ERROR" != "healthcheck" |~ "(timeout|deadline)"

# CIDR filter on extracted IP
{app="api"} | json | client_ip = ip("10.0.0.0/8")

# JSON parse + numeric filter on extracted field
{app="api"} |= "request" | json | status >= 500 | duration > 250ms

# logfmt parse (faster than JSON for flat key=value)
{app="grafana"} | logfmt | level="error"

# Pattern parse (much faster than regex for predictable formats; no backtracking)
{app="nginx"} | pattern `<ip> - <_> [<_>] "<method> <path> <_>" <status> <bytes>`

# Regex parse (last resort, slowest)
{app="legacy"} | regexp `(?P<ip>\S+) - (?P<user>\S+)`

# Filter on structured metadata (no parser needed; bloom-accelerated in 3.3+)
{app="api"} | trace_id="abc123def456789..."

# Reformat the displayed line
{app="api"} | json | line_format "{{.method}} {{.path}} -> {{.status}} ({{.duration}})"

# Promote extracted field to label
{app="api"} | json | label_format service=app_name

# Drop noisy labels from output
{app="api"} | drop level, host, container


# === METRIC QUERIES (PromQL-shaped) ===

# Logs per second per service
sum by (service) (rate({app="api"}[5m]))

# Error log rate per service
sum by (service) (rate({app="api", level="error"}[5m]))

# Error ratio (logs as metrics)
sum(rate({app="api", level="error"}[5m]))
  / sum(rate({app="api"}[5m]))

# Latency p99 from extracted duration field
quantile_over_time(0.99,
  {app="api"} |= "request" | json | unwrap duration [5m]
) by (service)

# Bytes shipped per service
sum by (service) (
  bytes_over_time({app="api"}[5m])
)

# Silent-failure alert: no logs in last 5 minutes
absent_over_time({app="critical-service"}[5m])

# Top 10 noisiest namespaces by line rate
topk(10, sum by (namespace) (rate({}[5m])))


# === DIAGNOSTIC COMMANDS (logcli) ===

# Cardinality audit -- the first command on any new cluster
logcli series '{}' --analyze-labels --since=1h

# Latest 100 log lines from a stream
logcli query --tail '{app="api", env="prod"}'

# Tail with line filter
logcli query --tail '{app="api"} |= "ERROR"'

# Count over time for a query
logcli query 'sum(rate({app="api"}[5m]))' --since=1h
```

---

## The 10 Commandments of Loki

1. **Thou shalt index labels, not log content.** Spine sticker, not full-text Google. If you find yourself wanting to "search every log line for X across 90 days," reach for Loki's structured metadata + bloom filters first; reach for ES only if the answer is genuinely no.

2. **Thou shalt never put `request_id`, `trace_id`, `user_id`, or `pod_name` in a label.** They go in structured metadata (3.0+) or the log line. Run `logcli series '{}' --analyze-labels --since=1h` weekly.

3. **Thou shalt run exactly one active compactor per tenant per cluster.** Two corrupt the index. Use leader election via `compactor_ring: { kvstore: { store: memberlist } }` and run 2-3 replicas for HA.

4. **Thou shalt always enable the WAL on ingesters.** A pod restart without WAL loses 30min-2h of un-flushed data per pod. `ingester.wal.enabled: true` and a PVC that survives the restart.

5. **Thou shalt place line filters BEFORE parsers.** `{} |= "ERROR" | json` is dramatically faster than `{} | json | line=~"ERROR"` (5x-100x by selectivity). Code-review your team's recurring queries.

6. **Thou shalt set `ingestion_rate_strategy: global`.** The default `local` means rate limits are per-distributor; a fleet of 5 distributors gives 5x the rate. Almost certainly not what you want.

7. **Thou shalt enable multi-tenancy from day one.** `auth_enabled: true` plus a real tenant name. Migrating off `fake` later is painful. Even single-team clusters benefit from a real tenant name.

8. **Thou shalt put `traceID` in structured metadata, not labels.** Then wire `tracesToLogsV2` in the Tempo data source and `derivedFields` in the Loki data source. The bidirectional click loop is the marquee feature of the entire observability stack.

9. **Thou shalt migrate off Promtail before March 2, 2026.** It's EOL. `alloy convert --source-format=promtail` is the migration path. **Grep your alerts for `promtail_*` first** -- the metric-name rename to `loki_source_file_*` / `loki_process_*` / `loki_write_*` is silent breakage.

10. **Thou shalt size for stream churn, not just steady-state cardinality.** 100k active streams is the guideline. A K8s cluster with `pod_name` in labels and 10 deploys/day will create millions of streams over the retention window even if the active count looks fine. WAL replay OOMs on cumulative cardinality, not current.

---

## Why Log Aggregation with Loki Matters Strategically

Logs were the last of the three pillars to get a cost-disruptive, OTel-native, K8s-shaped answer. Metrics had Prometheus since 2013; traces had Jaeger and then Tempo; logs spent a decade choosing between the operational complexity of Elasticsearch and the cost ceiling of "tail container logs to CloudWatch and grep them in the console." Loki bent the cost curve hard enough that **30 days of multi-tenant logs at 1 TB/day is a tractable budget for a mid-sized org**, which it explicitly was not on ES. That cost shift is the reason "log everything that matters and keep it for at least a month" became operationally normal in the late 2020s.

The other shift is the unification with traces. Until structured metadata arrived in Loki 3.0 (April 2024), the trace-to-logs pivot was theoretical -- the cardinality math made it practically impossible to put trace_id in labels, and stuffing it in the log body made queries slow. Structured metadata + bloom filters in 3.3+ made the click-through fast enough that engineers actually use it during incidents, which is what closes the three-signal loop the entire observability week was building toward.

The single most consequential decision you'll make in a Loki rollout is **the cardinality discipline of your collector config** -- which `__meta_*` labels get promoted to real labels, which fields go to structured metadata, which stay in the log body. Get it right and Loki is cheap, fast, and pleasant to operate for years. Get it wrong and you're in a permanent firefight with OOM ingesters, exploding S3 file counts, and slow queries that nobody trusts. Spend the time on the collector config; everything else flows from it.

---

## Further Reading

- [Loki Architecture](https://grafana.com/docs/loki/latest/get-started/architecture/) -- the canonical write-path/read-path component reference
- [Loki Deployment Modes](https://grafana.com/docs/loki/latest/get-started/deployment-modes/) -- monolithic vs SSD vs microservices decision framework
- [Loki Label Best Practices](https://grafana.com/docs/loki/latest/get-started/labels/bp-labels/) -- the cardinality discipline rules
- [Loki Cardinality Documentation](https://grafana.com/docs/loki/latest/get-started/labels/cardinality/) -- the math and the diagnostic commands
- [LogQL Log Queries Reference](https://grafana.com/docs/loki/latest/query/log_queries/) -- every parser, filter, formatter
- [LogQL Metric Queries Reference](https://grafana.com/docs/loki/latest/query/metric_queries/) -- range aggregations and `unwrap`
- [Loki 3.0 Release Blog](https://grafana.com/blog/2024/04/09/grafana-loki-3.0-release-all-the-new-features/) -- structured metadata, native OTLP, original bloom filters
- [Loki 3.3 Bloom Filters Pivot](https://grafana.com/blog/2024/11/21/grafana-loki-3.3-release-faster-query-results-via-blooms-for-structured-metadata/) -- the bloom-on-structured-metadata redesign
- [Loki 3.3 Release Notes](https://grafana.com/docs/loki/latest/release-notes/v3-3/) -- breaking changes including the bloom-compactor removal
- [Promtail to Alloy Migration Guide](https://grafana.com/docs/alloy/latest/set-up/migrate/from-promtail/) -- the official `alloy convert` walkthrough
- [Promtail to Alloy 2025 Field Report](https://developer-friendly.blog/blog/2025/03/17/migration-from-promtail-to-alloy-the-what-the-why-and-the-how/) -- production-grade Helm values and the metric-rename gotcha
- [Grafana Alloy Documentation](https://grafana.com/docs/alloy/latest/) -- River syntax and component reference
- Repo cross-references: [Distributed Tracing Deep Dive](../opentelemetry/2026-04-28-distributed-tracing-deep-dive.md), [OpenTelemetry Fundamentals](../opentelemetry/2026-04-27-opentelemetry-fundamentals.md), [Kubernetes Metrics Stack](../prometheus/2026-04-26-kubernetes-metrics-stack.md), [Grafana Fundamentals](../grafana/2026-04-24-grafana-fundamentals.md), [Alertmanager & Alerting Strategy](../prometheus/2026-04-23-alertmanager-alerting-strategy.md), [PromQL Deep Dive](../prometheus/2026-04-21-promql-deep-dive.md), [Prometheus Fundamentals](../prometheus/2026-04-20-prometheus-fundamentals.md)
