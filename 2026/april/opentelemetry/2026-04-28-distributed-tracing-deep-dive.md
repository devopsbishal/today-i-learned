# Distributed Tracing Deep Dive -- ATC Replay for Microservices

> Yesterday's OpenTelemetry Fundamentals doc covered the API/SDK/Collector split, OTLP wire format, K8s deployment patterns, and lightly introduced sampling and the trace data model. Today goes deep on the trace signal itself: the byte-level mechanics of W3C Trace Context propagation, the math of head vs tail sampling, the architecture of a tail-sampling Collector tier (and why the `loadbalancing` exporter is non-negotiable), Tempo's columnar object-storage backend, TraceQL's structural query language, the spanmetrics connector that closes the trace-to-metrics loop, and the production failure modes that show up in interviews and post-mortems.
>
> The central analogy for the day: **distributed tracing is air-traffic-control replay for microservices.** When a 737 disappears off the radar over the Atlantic, no single artifact tells investigators what happened. They need three things, all stitched together by a single shared identifier (the flight number plus tail-number plus UTC timestamp): the **cockpit voice recorder** (what the crew said inside the aircraft), the **flight data recorder** (what every instrument and control surface did), and the **ATC tape** (every controller hand-off as the plane crossed FIRs). Each artifact alone is fragments. Together they reconstruct the flight minute by minute, hand-off by hand-off, decision by decision. **A trace is the FDR + CVR + ATC tape for one user request as it crosses your services.** Each span is one segment of the flight: take-off (root SERVER span at the ingress), each ATC hand-off (a CLIENT span calling the next service), each in-flight event (span events: turbulence, autopilot disengaged, exception thrown), and the final landing or crash (span status `OK` or `ERROR`). The `traceparent` HTTP header is the **flight strip** every controller passes to the next sector -- the same flight identity, propagated across boundaries, so the recorders all stamp the same flight number. The reason every airline accident investigation in the last 50 years has resolved is that this stitching is *mandatory and standardized*; the reason most production incidents in 2018 were unsolvable was that everyone was running their own private flight recorder with no common tail number. OTel's W3C Trace Context is what finally gave microservices the standardized flight-strip format.
>
> Two corollaries the analogy carries: **sampling is the air-accident-investigation budget** -- you don't replay every uneventful flight, but you absolutely keep every crash recording and every recorder from a flight that diverted. The math of head sampling ("flip a coin at the gate") vs tail sampling ("look at the whole flight, then decide") is exactly this. And **the spanmetrics connector is the FAA's monthly safety statistics report** -- aggregate counts and percentiles derived from the same recorders that fed individual investigations, with exemplar links back to specific flights when one stat looks weird. The same recorders, two consumers.

---

**Date**: 2026-04-28
**Topic Area**: opentelemetry
**Difficulty**: Advanced

---

## TL;DR -- Concepts at a Glance

| Concept | Real-World Analogy | One-Liner |
|---------|-------------------|-----------|
| Trace | Full flight recording (CVR + FDR + ATC tape) for one request | Tree of spans sharing a `trace_id` |
| Span | One segment of the flight, between two ATC hand-offs | Unit of work with `name`, `kind`, `start`, `end`, `attributes`, `events`, `parent_span_id` |
| `trace_id` | The flight number + tail number + UTC timestamp combined | 16-byte hex; the join key across spans, logs, metrics, exemplars |
| `span_id` | The segment ID inside one flight recording | 8-byte hex; identifies one span uniquely within a trace |
| Span kind | The role of this segment (departure / cruise / hand-off / arrival) | `SERVER` / `CLIENT` / `PRODUCER` / `CONSUMER` / `INTERNAL` -- gets the service map right or wrong |
| Span events | Time-stamped notes in the cockpit log within one segment | Sub-events inside a span: exceptions, retries, GC pauses |
| Span links | "See also" cross-reference to a different flight | Cross-trace pointers for batch / fan-in / async |
| Span status | "Landed normally" / "Diverted" / "Crashed" | `Unset` / `Ok` / `Error` -- what flips a span red |
| W3C Trace Context | The flight strip every ATC controller hands the next sector | `traceparent: 00-<trace_id>-<span_id>-<flags>` HTTP header |
| `tracestate` | Vendor-specific addenda the controllers staple onto the strip | Multi-vendor sidecar header, 32-entry cap, most-recent-first |
| Baggage | The flight crew's clipboard notes (NOT the flight strip) | Userland key-value carrier; propagates across services but is *not* trace context |
| Head sampling | Decide at the gate: "is this flight worth recording?" | Stateless, parent-propagated, cheap; loses errors that emerge later |
| Tail sampling | Wait until the flight lands, then decide whether to keep the tape | Stateful, sticky-routed, requires `loadbalancing` exporter; catches errors |
| `loadbalancing` exporter | A sticky seating chart that always sends the same flight to the same investigator | Hashes by `trace_id`, routes to the same downstream Collector replica every time |
| `decision_wait` | The black-box recovery window | How long the tail sampler buffers spans before deciding -- must exceed your slowest p99.9 |
| Spanmetrics connector | The FAA's monthly safety statistics report, derived from recorders | Connector emitting RED metrics from spans, with exemplars pointing back to traces |
| Exemplars | The "click for the actual flight recording" link on a stat | A `trace_id` attached to one observation in a histogram bucket |
| Service graph | The world airline route map, reconstructed from ATC hand-off pairs | Edges built from CLIENT/SERVER span pairs across services |
| Tempo | A specialized aviation-archive that stores recordings as columnar blocks on cheap object storage | Object-storage-backed trace backend; no Cassandra/ES, S3/GCS only |
| TraceQL | The investigators' structured query language for the recording archive | Tempo's query language with span selectors, structural operators (`>>`, `>`), aggregates |
| Jaeger | The traditional Cassandra/Elasticsearch-backed flight archive | Tag-search trace backend; predates Tempo's structural-query model |
| AWS X-Ray | AWS's in-house aviation archive with built-in sampling rules | Managed trace backend reachable from OTel via ADOT or the `awsxray` exporter |

---

## The Big Picture -- One Trace, Three Recorders, Five Decisions

```
DISTRIBUTED TRACING -- END-TO-END VIEW
================================================================================

  USER REQUEST                                        TRACE BACKEND (Tempo)
  +----------+                                        +-----------------------+
  |  client  |                                        |  Object storage (S3)  |
  +----+-----+                                        |  + columnar blocks    |
       |                                              |  + bloom filters      |
       | HTTP POST /checkout                          |  + indexes            |
       | traceparent: 00-a1b2..-001..-01              +-----------+-----------+
       v                                                          ^
+--------------+  CLIENT span                                     |
|  ingress     |--+                                          OTLP queries
+------+-------+  |                                               |
       | SERVER   | traceparent rewritten on egress     +---------+---------+
       v          |   00-a1b2..-002..-01                |       Tempo       |
+--------------+  |                                     |    distributors,  |
|  checkout svc|--+                                     |    ingesters,     |
+--+---------+-+                                        |    queriers       |
   |         |                                          +---------^---------+
   |         | (PRODUCER -> Kafka)                                |
   v         |                                                    |  OTLP
+----+----+  |                                          +---------+---------+
| auth svc|  |                                          |   GATEWAY OTel    |
+----+----+  |                                          |   COLLECTOR POOL  |
     |       |                                          |                   |
     |       v       (CONSUMER, linked, not child)      |  loadbalancing    |
     |  +-------------+         +------------------+    |  -> tail_sampling |
     |  | inventory   |         | order-fulfiller  |    |  -> spanmetrics   |
     |  +-------------+         +------------------+    |  -> tempo, mimir  |
     |                                                  +---------^---------+
     v                                                            |
                                                                  | OTLP
                                                                  |
                                                       +----------+---------+
                                                       |  AGENT DAEMONSET   |
                                                       |  per-node Collect. |
                                                       +----------^---------+
                                                                  |
                                                                  | OTLP
                                                                  | localhost:4317
                                                                  |
                                                            APP PODS (SDK)

      ALL SPANS CARRY trace_id = a1b2c3...                FLIGHT STRIP:
      ALL LOGS CARRY trace_id = a1b2c3...                  traceparent
      ALL EXEMPLARS CARRY trace_id = a1b2c3...             header on every
                                                            outbound RPC
```

Five things this picture shows that are easy to miss:

1. **The same `trace_id` flows through HTTP, Kafka, gRPC, and back into HTTP.** Every transport carries the trace context in its native header location. HTTP/gRPC: `traceparent` header. Kafka: a record header. AWS SQS: a message attribute. In every case the SDK's *propagator* is what reads it on ingress and writes it on egress.
2. **Span kind shapes the service graph.** The edge from `checkout` to `auth` exists in the service map only because checkout emits a CLIENT span and auth emits a SERVER span with the matching parent ID. Get the kind wrong and the edge is invisible -- not "wrong," literally not drawn.
3. **The Kafka boundary is async, so the consumer span is *linked* to the producer span, not parented under it.** The trace would otherwise have to "wait" hours of queue time. The link preserves the relationship without forcing the tree to span the queue's storage time.
4. **Tail sampling is the gateway pool's job, and `loadbalancing` is what makes it work.** A trace's spans arrive at the gateway from multiple agents at different times; the consistent-hash-by-trace-id ring guarantees they all converge on the same gateway replica's in-memory buffer.
5. **The spanmetrics connector consumes spans inside the gateway and emits derived metrics.** Same recorders, two outputs: the raw span stream goes to Tempo; the aggregated `calls_total` and `duration_seconds_bucket` go to Mimir, with exemplars pointing back to specific trace IDs.

---

## Part 1: Span Anatomy -- The Black Box of One Flight Segment

A span is the atomic unit of a trace. Yesterday's doc listed the fields; today we go field by field with the implications most teams discover only after their service map is wrong or their alert never fires.

### The span structure on the wire

```
Span:
  trace_id:        16 bytes (32 hex chars), the flight number
  span_id:         8 bytes (16 hex chars),  the segment ID
  parent_span_id:  8 bytes or null (root)
  trace_state:     vendor sidecar, optional
  flags:           sampled bit + others
  name:            short, low-cardinality label
  kind:            SERVER | CLIENT | PRODUCER | CONSUMER | INTERNAL
  start_time_unix_nano: 19 digits typically
  end_time_unix_nano:   19 digits typically
  attributes:      map<string, AnyValue>
  events:          [{timestamp, name, attributes}]
  links:           [{trace_id, span_id, attributes}]
  status:          {code: Unset|Ok|Error, message: string}
  resource:        {service.name, k8s.*, cloud.*, ...}
```

### Span name -- low-cardinality always

The name is what shows up as the span title in the trace waterfall and as the `span_name` label on spanmetrics-derived metrics. Cardinality discipline is identical to Prometheus labels (Apr 20):

```
GOOD:  POST /api/users/:id
       checkout.submit
       db.query SELECT users
       redis SETNX

BAD:   POST /api/users/12345
       checkout submitted at 2026-04-28T13:42:11
       SELECT * FROM users WHERE id=12345
```

The route template (`:id`) goes in the name; the actual ID goes in the `http.route` and `user.id` attributes if needed. Auto-instrumentation gets this right by default for HTTP and DB; manual instrumentation is where teams blow it up.

### Span kind -- the field that builds (or breaks) your service map

There are five kinds. The dependency-graph implications matter:

| Kind | Meaning | Where to set it | Service-graph implication |
|------|---------|-----------------|---------------------------|
| `SERVER` | This service received a request and is handling it | The root of each service's segment of the trace | A node in the service graph; the *receiving* end of an edge |
| `CLIENT` | This service is calling another service | Wrapping HTTP/gRPC/DB calls outbound | The *sending* end of an edge; pairs with the downstream SERVER span |
| `PRODUCER` | This service is enqueuing onto an async medium (Kafka, SQS, RabbitMQ) | Wrapping `producer.send()` | A node that emits to a queue topic |
| `CONSUMER` | This service is dequeuing from an async medium | Wrapping `consumer.poll()` handler | A node that reads from a queue topic; usually *linked* to the producer span, not parented |
| `INTERNAL` | This is a sub-step inside one service, not a network call | Wrapping a business operation, an internal computation | Not drawn on the service graph at all |

The single most damaging silent failure: **using `INTERNAL` for a span that was actually a network call.** The service graph misses the edge entirely; downstream service appears disconnected; you spend weeks wondering why the dependency map is missing critical links. Auto-instrumentation gets this right. Hand-rolled spans wrapping a `requests.get()` and tagging it `INTERNAL` because "it's just a function call from my perspective" is the canonical bug.

The matching rule for service graphs:

```
edge(serviceA -> serviceB) exists iff
  span_A.kind = CLIENT
  AND span_B.kind = SERVER
  AND span_B.parent_span_id = span_A.span_id
  AND span_A.resource.service.name = serviceA
  AND span_B.resource.service.name = serviceB
```

Both sides need to be right. If the upstream tags its span `INTERNAL` -- no edge. If the downstream tags its span `CLIENT` (because it thinks it's calling someone) -- no edge.

PRODUCER/CONSUMER pairs differ: the consumer is typically a *link* back to the producer, not a child, so the edge is drawn from the link, not the parent_span_id.

### Span attributes -- key-values, with cardinality discipline

Attributes are free-form key-value pairs:

```python
span.set_attribute("http.method", "POST")
span.set_attribute("http.route", "/api/users/:id")
span.set_attribute("http.status_code", 500)
span.set_attribute("user.tier", "enterprise")
span.set_attribute("checkout.cart_size", 7)
```

Three rules earn their keep:

1. **Use semantic-convention names.** `http.status_code` not `httpStatus`. `db.system` not `database_type`. The OTel spec ([semconv](https://opentelemetry.io/docs/specs/semconv/)) is the canonical list. Tempo's TraceQL queries you'll see in Part 6 assume these names; deviation costs you query reusability.
2. **Tempo doesn't enforce attribute cardinality the way Prometheus does.** Putting `user.id` (a million unbounded values) on a span attribute is fine for the *trace* signal -- Tempo stores attributes in columnar blocks, not as indexed labels. But:
3. **The spanmetrics connector turns attributes into metric labels.** If you list `user.id` in spanmetrics' `dimensions: [...]` config, every unique user produces a new metric series. Cardinality bomb. The discipline: attributes you might *put on a metric* are bounded; attributes that are forensic (high-cardinality, queryable in TraceQL) stay off the spanmetrics dimension list.

### Span events -- timestamped notes in the cockpit log

An event is a sub-record inside a span at a specific timestamp:

```python
span.add_event(
    "cache.miss",
    attributes={"cache.key": "user:42", "cache.backend": "redis"},
)
# ...
try:
    charge_payment(...)
except PaymentDeclined as e:
    span.record_exception(e)  # adds an "exception" event with stack trace
    span.set_status(StatusCode.ERROR, str(e))
```

The `exception` event is the canonical use: `span.record_exception(e)` adds an event named `exception` with `exception.type`, `exception.message`, and `exception.stacktrace` attributes attached to the *event*, not the span. This is why a stack trace appears at a *specific timestamp inside the span timeline* in Grafana's Tempo waterfall, not at the start.

Other events that earn their keep:

- `cache.miss`, `cache.hit` for fine-grained cache instrumentation when you don't want to start a child span for each lookup.
- `retry.attempt` to mark each retry inside one logical span.
- `gc.pause` if you're doing JVM-level instrumentation and want the pause visible inline.
- `feature_flag.evaluated` to record which flag value the request observed.

The decision rule between **events vs child spans** comes up often:

| Use a span event when | Use a child span when |
|----------------------|------------------------|
| The sub-step is fast (< 100us) and you don't need duration | The sub-step takes meaningful time and you want it on the waterfall |
| You don't need attributes the parent span doesn't have | The sub-step has a meaningfully different attribute set |
| The thing happens at a single instant | The thing has a start and an end you care about |
| Cache hit/miss markers | Cache lookup that hit the network |

Span events are cheap (they don't multiply trace volume by N); child spans are richer but add to the span count, which directly drives ingest cost and tail-sampling buffer size.

### Span status -- Unset, Ok, Error -- and why Unset matters

Three values:

| Status | When |
|--------|------|
| `Unset` | The span ended without any explicit status decision. **This is the default.** Auto-instrumentation usually sets `Ok` only when explicitly required by the spec; most successful spans actually carry `Unset`. |
| `Ok` | The span ended successfully and the SDK was told to mark it explicitly. Usually only the case when an instrumentation libary or the application code calls `set_status(Ok)`. |
| `Error` | The span ended with an error. Either: the application called `set_status(Error)`; auto-instrumentation detected an exception or an HTTP 5xx; the span recorded an exception event. |

Why `Unset` matters: the OTel spec says **a span that ends without `Error` should be treated as successful**, but it does *not* require the SDK to set `Ok` explicitly. This means a TraceQL filter like `{ status = ok }` will *miss most successful spans* because their status is `Unset`. The correct filter for "successful spans" is `{ status != error }`. Tempo's docs warn about this; teams that learn it the hard way usually have an alert that fires "no successful traffic" because of an `= ok` filter.

Don't conflate latency with errors:

- A span that took 30 seconds but completed = `Unset` (or `Ok`).
- A span that took 100ms and threw an exception = `Error`.

A slow span isn't an error. The tail-sampler's *latency* policy handles "keep slow traces"; the *status_code* policy handles "keep error traces." They're orthogonal.

### Span links -- "see also" cross-references

A `parent_span_id` says "this span is a step inside that span's work." A *link* says "this span is related to that other span but not nested under it." Yesterday's doc covered the four legitimate cases; one more from a tracing-deep-dive lens:

The most common interview question: **"How do you trace a Kafka producer/consumer flow?"** The right answer:

1. Producer service starts a CLIENT span when calling `producer.send()`. The span emits an `INTERNAL` (or `PRODUCER`) sub-span recording "submitted message to topic X."
2. Producer service's propagator writes the trace context into the Kafka *record headers* (`traceparent` is the convention; OTel's Kafka instrumentation does this automatically).
3. Consumer service starts a CONSUMER span when processing the message. Critical decision: this span's `parent_span_id` is **null** (it's a root span in a new trace), but it has a `link` pointing at the producer's PRODUCER span via that span's `trace_id` and `span_id`.
4. The trace is now two separate trees connected by a link. Tempo's query "show me all traces linking to trace X" finds both halves.

The reason for this shape rather than parenting: a trace tree should not span hours of queue dwell time. A 50ms producer span being the parent of a CONSUMER span 6 hours later breaks every duration aggregation, every span-buffer-window sampling decision, and every visual timeline. Two traces linked is the cleaner model.

---

## Part 2: W3C Trace Context -- The Flight Strip Format

Every modern tracer speaks W3C Trace Context. It's the wire-level standard for propagating trace identity across HTTP/gRPC boundaries. Memorize the format byte by byte.

### `traceparent` -- the only header that matters

The format:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
              ^^  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^  ^^
              |   |                                 |                  |
              |   |                                 |                  +-- flags (2 hex)
              |   |                                 +-- parent-id      (16 hex = 8 bytes)
              |   +-- trace-id                                          (32 hex = 16 bytes)
              +-- version                                               (2 hex; "00" today)
```

Four fields, dashes between, lowercase hex:

| Field | Bytes | Notes |
|-------|-------|-------|
| `version` | 1 byte (`00`) | Always `00` in 2026; future versions must be back-compat |
| `trace-id` | 16 bytes | All-zero is invalid; uniformly random in practice. **Same across all spans of one trace.** |
| `parent-id` | 8 bytes | The `span_id` of the **caller's** span -- this is what becomes the new span's `parent_span_id` on the receiving service |
| `flags` | 1 byte | Bitfield. The only widely respected bit is **`01` = sampled** |

The `flags=01` (sampled) bit is consequential:

- A service receiving a request with `flags=01` **may honor the sampling decision** -- the W3C spec frames this as a recommendation rather than a strict mandate (callers can lie or DoS you with `flags=01` everywhere; the receiver is allowed to override). In practice, `parentbased_traceidratio` -- the production-default sampler -- *chooses* to honor the parent flag, and that's what makes head sampling consistent across services.
- `flags=00` means "the upstream did not record this." A `parentbased_traceidratio` sampler will then make an independent decision based on its configured ratio; a strict `parentbased_alwayson` would drop the unsampled-by-parent case.
- The SDK reads `flags` on ingress (in the `extract` step of the propagator) and writes them on egress (in the `inject` step) so the bit propagates downstream.

The other 7 bits of `flags` are reserved and must be ignored if not understood.

### `tracestate` -- the vendor-multiplexed sidecar

```
tracestate: dd=s:2;p:00f067aa0ba902b7,vendor2=t61rcWkgMzE
```

Format: comma-separated key-value pairs, each pair is `vendorKey=vendorValue`. Used by vendors to pass vendor-specific state along the same propagation chain. Caps and rules:

- **Maximum 32 entries.**
- **Mutated entries should move to the left.** Per the spec: unmodified entries' order *must* be preserved; modified keys *should* be moved to the leftmost position. ("Should," not "must" -- some implementations don't reorder, and that's spec-compliant.)
- **Truncation is two-step, not pure "drop oldest."** When the size cap is exceeded, the spec says: first, entries longer than 128 characters *should* be removed; if still over the limit, entries *should* be removed starting from the rightmost (the oldest). The size-based eviction comes first.
- **Vendor keys are namespaced by the vendor.** `dd=...` for Datadog, `congo=...` for the W3C example vendor. Custom keys have to be registered.
- **Total length cap is 512 chars** in the original spec; backends often enforce stricter limits.

A common mistake: cramming application data into `tracestate`. Don't. Use **Baggage** (next).

### `baggage` -- userland key-value, NOT trace context

`baggage` is a separate W3C standard ([W3C Baggage](https://www.w3.org/TR/baggage/)) for propagating userland key-value pairs across service boundaries. It rides as its own HTTP header:

```
baggage: userId=abc123,sessionTier=premium,featureFlag.checkout_v2=on
```

The single most common point of confusion: **Baggage is not trace context.** Baggage carries application-scoped key-value pairs; trace context carries the trace identity. A service can extract Baggage and use it to set span attributes:

```python
from opentelemetry import baggage

# Receiving service: read baggage from incoming request
user_id = baggage.get_baggage("userId")
span.set_attribute("user.id", user_id)
```

The SDK's `Propagators` is what handles both -- it's a composite that runs `traceparent` + `tracestate` + `baggage` extractors/injectors in order. Configuration:

```bash
# Yesterday's Instrumentation CR set this:
OTEL_PROPAGATORS=tracecontext,baggage
```

Three propagators in common deployment:

| Propagator name | What it handles |
|-----------------|-----------------|
| `tracecontext` | W3C Trace Context (`traceparent`, `tracestate`) |
| `baggage` | W3C Baggage (`baggage`) |
| `b3` or `b3multi` | Zipkin's B3 headers (`X-B3-TraceId`, `X-B3-SpanId`, `X-B3-Sampled` etc.); needed if you have legacy Zipkin/Sleuth services in the mix |
| `jaeger` | Jaeger's `uber-trace-id` header; legacy Jaeger fleet |
| `xray` | AWS X-Ray's `X-Amzn-Trace-Id`; needed when traces traverse AWS-native services like API Gateway, ALB, Lambda |

The OTel SDK runs **all configured propagators in sequence** on both extract and inject. If you have any legacy or AWS service in the path, you usually want a multi-propagator config:

```bash
OTEL_PROPAGATORS=tracecontext,baggage,b3multi,xray
```

The OTel Operator's `Instrumentation` CRD sets this via the `propagators:` list (yesterday's Part 9), and the auto-instrumentation init container drops the env var into your pod.

### B3 vs W3C -- the legacy comparison

B3 (Zipkin's format) was the de-facto standard pre-2020. The differences:

| Aspect | W3C Trace Context | B3 |
|--------|-------------------|----|
| Header(s) | Single: `traceparent` (+ optional `tracestate`) | Multi: `X-B3-TraceId`, `X-B3-SpanId`, `X-B3-ParentSpanId`, `X-B3-Sampled` (or single `b3:` header in `b3` mode) |
| Trace ID size | 16 bytes only | 8 or 16 bytes (legacy 8-byte support causes interop pain) |
| Sampled flag | Top bit of `flags` byte | `X-B3-Sampled: 1` separate header |
| Vendor sidecar | `tracestate`, multi-vendor | None; B3 is single-tenant in spirit |
| Status as of 2026 | Standard, mandated by OTel default | Legacy; supported via `b3` propagator for compat |

The migration path for an organization with a mixed fleet: configure `OTEL_PROPAGATORS=tracecontext,baggage,b3multi`. New services emit `traceparent`; legacy services still send and accept B3. The OTel SDK handles the translation seamlessly. Once all services are upgraded, drop `b3multi`.

### The ingress-controller mutation gotcha

A subtle production failure: NGINX Ingress, AWS ALB, and some service meshes (older Istio versions) **rewrite or strip the `traceparent` header by default**. The trace fragments at the ingress boundary and you get two disconnected halves. Fixes:

- **NGINX Ingress**: add `nginx.ingress.kubernetes.io/configuration-snippet` to copy `traceparent` and `tracestate` to upstream, OR enable the OpenTelemetry NGINX module which generates its own SERVER span and propagates correctly.
- **AWS ALB**: ALB does not natively propagate `traceparent`; it injects `X-Amzn-Trace-Id`. The fix is to configure the OTel `xray` propagator alongside `tracecontext`, or bridge the two in the Collector via a `transform` processor.
- **Istio / Envoy**: 1.12+ supports W3C Trace Context natively; older versions need explicit `tracing.zipkin` config and B3 headers.

Diagnostic: every trace's root span is the *ingress*, not the app. If the app is showing as a root span, the ingress is stripping the header.

---

## Part 3: Sampling -- The Air-Accident-Investigation Budget

You cannot record every flight. The math doesn't work: at scale, full trace ingest costs more than the rest of your observability stack combined, and the value-per-trace decays steeply (the 100,000th identical successful checkout teaches you nothing). Sampling is how you spend the budget.

The interview-defining question: **"Walk me through how you'd configure sampling for a 200-service production fleet at 50k requests/sec."** There is no single right answer; there is a correct *framework* for arriving at one.

### Head sampling -- decide at the gate

**Head sampling decides at trace start whether to record.** The decision is propagated downstream via the `flags=01` bit on `traceparent`. Two flavors:

**1. `traceidratio`** -- a bare percentage, applied at every service independently. If service A samples at 10% and service B samples at 5%, every service's spans get its own coin flip and traces are randomly fragmented.

**2. `parentbased_traceidratio`** (the production default) -- if the parent's `flags=01`, this service samples; if `flags=00`, this service uses the configured ratio. The first service in the chain (the ingress / root) makes the only real decision; everyone else honors it. **Result: a sampled trace is fully sampled; an unsampled trace is fully unsampled.** This is what makes head sampling *consistent*.

Why sample by trace ID hash and not by random number: the SDK computes the sampling decision deterministically from the `trace_id` itself:

```
sampled = (lowest_8_bytes_of_trace_id_as_uint64 / 2^64) < ratio
```

Two services that haven't communicated yet will agree on whether a given `trace_id` is sampled, *as long as both services use the same sampler implementation with the same ratio*. This is what makes head sampling work even when propagation gets dropped at a boundary -- a downstream service that lost the parent context can re-derive the sampling decision from the trace ID alone. (`traceidratio` does this; `parentbased_traceidratio` adds the parent-flag-honor on top.) Mismatched ratios across services give you fragmented traces where some services sampled and others didn't -- a real production trap when teams own their own SDK config without coordination.

The math:

```
At 1% head sampling, 50k req/sec:
  500 sampled traces/sec
  ~5 spans/trace average
  ~2,500 spans/sec ingested
  ~2 KB/span on the wire (post-batching, OTLP/protobuf)
  -> 5 MB/sec -> ~430 GB/day raw OTLP volume
  After Tempo's columnar block compression (~5x): ~85 GB/day stored
  Tempo ingest cost: tractable
```

(The wire-vs-stored split matters for billing. Tempo's Parquet-style columnar blocks compress aggressively, but you pay ingest by the *uncompressed* OTLP byte rate, not the stored byte rate, on most managed offerings.)

**Head sampling pros**: stateless, cheap, deterministic, scales infinitely, no buffer memory cost.

**Head sampling con (the killer)**: errors and slow traces are not visible at trace start. By the time a 99th-percentile latency emerges, the sampling decision has already been made. **Errors that occur in head-sampled-out traces are gone forever.** The on-call engineer searching for a 5xx trace at 3am has a 99% chance of finding nothing, because the 99% of unsampled traces is exactly where the 5xx happened.

This is why pure head sampling is operationally inadequate above hobbyist scale.

### Tail sampling -- wait for the flight to land

**Tail sampling decides after seeing the entire trace whether to keep it.** It runs in the Collector, not the SDK. The SDK is configured to emit *all* traces (sampling rate = 100%); the Collector batches the spans, waits `decision_wait` for the trace to complete, applies policies, then keeps or drops.

The architecture:

```
[Apps emit 100% of traces]
        |
        v  OTLP, all spans
+----------------------------+
| AGENT DAEMONSET (per-node) |  -- fast forward, no sampling decision
+----------------------------+
        |
        v  OTLP, all spans, batched by trace_id at the next hop
+--------------------------------+
| FRONT GATEWAY                  |
| `loadbalancing` exporter       |  -- consistent-hashes by trace_id
+--------------------------------+
        |     |     |
        v     v     v
+------+   +------+   +------+
| TAIL |   | TAIL |   | TAIL |    -- StatefulSet pool, each replica
| pool |   | pool |   | pool |       buffers its hash-shard's
| rep0 |   | rep1 |   | rep2 |       traces for `decision_wait`
+--+---+   +--+---+   +--+---+
   |          |          |
   +----------+----------+
              |
              v  OTLP, sampled traces only
       +-----------+
       |   Tempo   |
       +-----------+
```

Three load-bearing components:

1. **The `loadbalancing` exporter** routes by **consistent hash** on the trace ID so all spans of a trace land on the same downstream replica. Consistent hashing (not modulo-N) is what keeps the ring stable when replicas scale up or restart -- otherwise every replica change would re-shard every in-flight trace, fragmenting decisions. Without this exporter, span A of trace T might go to replica 0 and span B might go to replica 1 -- neither has the full trace, neither can sample correctly. The default `routing_key` for trace pipelines is `traceID` (the only sane choice for tail sampling); the exporter also supports `service`, `metric`, `resource`, `streamID`, `attributes` for other use cases (e.g. routing metrics by `service.name` for per-tenant Mimir backends).
2. **The tail-sampling pool is a StatefulSet** behind a *headless* Service so DNS returns all pod IPs (not the Service VIP). This was yesterday's Part 8 footnote and it's worth re-emphasizing: the consistent-hash ring stays stable only if pod identities are stable.
3. **`decision_wait`** is the trace buffer window. Spans are held in memory for this long before the decision. Memory cost is the punch line:

```
buffer_size_per_replica = (spans_per_second_total / replicas) * decision_wait
```

A pool with 3 replicas seeing 5,000 spans/sec total (1,000 traces/sec * 5 spans/trace) at 30s decision_wait, with each replica holding its 1/3 hash shard:

```
buffer_size_per_replica = (5,000 / 3) * 30 ~= 50,000 spans buffered per replica
```

At ~2 KB per span in OTel's internal in-memory representation (heap-resident protobufs are larger than the wire form -- factor in object headers and pointer overhead and you're closer to ~4 KB), that's ~200-400 MB per replica just for the tail-sampling buffer. Plus `num_traces` overhead (the policy state per trace ID). Real-world pools at 10x this volume run 2-8 GB per replica; the memory math scales linearly.

`decision_wait` floor: longer than your slowest service's p99.9. Set 30s as a typical default; raise it if any service routinely takes longer (long-poll endpoints, batch handlers, ML inference pipelines). Set it too short and slow traces are decided *before* their slow spans arrive, fragmenting the trace.

### The tail-sampling policies

The `tail_sampling` processor's policy catalog (from the [README](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/tailsamplingprocessor/README.md)):

| Policy type | What it samples |
|-------------|-----------------|
| `always_sample` | Every trace. Useful as a debugging fallback. |
| `probabilistic` | A configurable percentage of all traces. |
| `rate_limiting` | At most N traces/sec, regardless of volume. Caps cost. |
| `latency` | Traces whose total duration exceeds a threshold (e.g. > 1000ms). |
| `numeric_attribute` | Traces where some attribute is in a numeric range (e.g. `http.status_code >= 500`). |
| `boolean_attribute` | Traces where a boolean attribute matches. |
| `string_attribute` | Traces where a string attribute matches a list of values (e.g. `service.name in [checkout, payment]`). |
| `status_code` | Traces with at least one span whose status is in the listed codes (`OK`, `ERROR`, `UNSET`). The "keep all errors" policy. |
| `trace_state` | Traces whose `tracestate` matches a regex. Vendor-flag-driven sampling. |
| `trace_flags` | Traces with specific `traceparent` flag bits set (e.g., the `sampled` bit if you want to honor an upstream head decision). |
| `span_count` | Traces with a span count in a configured range. The "drop runaway traces" lever -- catches traces with thousands of spans (cardinality bombs from instrumentation bugs). Pairs directly with gotcha #3. |
| `ottl_condition` | An arbitrary [OTTL](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/pkg/ottl) expression evaluated against each span; the trace is kept if any span matches. The escape hatch when no built-in policy fits. |
| `composite` | A weighted combination of sub-policies with a rate limit per sub-policy. The "spend budget across categories" primitive. |
| `and` | Multiple policies that must *all* match for the trace to be kept. |
| `not` | Inverts a sub-policy. Useful with `and` to express "keep traces matching X *unless* they also match Y." |

The canonical production policy stack -- the one every senior engineer should be able to articulate from memory:

```yaml
tail_sampling:
  decision_wait: 30s
  num_traces: 100000              # max in-flight trace IDs to track
  expected_new_traces_per_sec: 2000

  policies:
    # 1. Errors: keep 100%, always
    - name: errors-100pct
      type: status_code
      status_code:
        status_codes: [ERROR]

    # 2. HTTP 5xx (covers errors that didn't set span status, common in older instrumentation)
    - name: http-5xx-100pct
      type: numeric_attribute
      numeric_attribute:
        key: http.status_code
        min_value: 500
        max_value: 599

    # 3. Slow: keep 100% of traces > 1s total duration
    - name: slow-100pct
      type: latency
      latency:
        threshold_ms: 1000

    # 4. Specific noisy services / endpoints sampled even harder down
    - name: healthcheck-drop-99pct
      type: and
      and:
        and_sub_policy:
          - name: is-health
            type: string_attribute
            string_attribute:
              key: http.route
              values: [/healthz, /readyz, /metrics]
          - name: drop-99
            type: probabilistic
            probabilistic:
              sampling_percentage: 1   # keep only 1% of healthchecks

    # 5. Critical-tenant traces always sampled (regulatory / VIP customer tier)
    - name: vip-tenant-100pct
      type: string_attribute
      string_attribute:
        key: tenant.tier
        values: [enterprise, regulated]

    # 6. Baseline -- everything else at 1%
    - name: baseline-1pct
      type: probabilistic
      probabilistic:
        sampling_percentage: 1
```

Three properties of this stack:

- **Policies are evaluated independently, results are OR'd.** A trace matches *any* policy -> kept. Errors and slow are kept at 100%, healthchecks at 1%, baseline at 1%, and tenancy overrides force critical traffic up.
- **Order doesn't matter for keep/drop logic.** It does matter for *attribution* in metrics emitted by the processor (which policy fired), but not for the decision.
- **`num_traces`** caps the in-flight trace ID set to bound memory; if it overflows, the oldest trace (by first-span-arrival time) is forcibly decided early. This is the back-pressure escape hatch.

### Head + tail: the production combo

Pure tail sampling at scale is expensive (every span buffered for 30 seconds). Pure head sampling at scale loses errors. The pragmatic answer:

**Tier 1 -- Head sample at SDK to a generous percentage** (e.g., 100% in dev, 25% in prod for low-volume services, 5-10% for the highest-traffic tier). This caps the SDK's emission rate.

**Tier 2 -- Tail sample at the gateway** with the policy stack above, applied to the spans that survived head sampling.

The math at 50k req/sec, head-sample 10%:

```
50k req/sec * 10% = 5k req/sec emitted by SDKs
5k * 5 spans/trace = 25k spans/sec at the agent layer
At gateway tail sampling, errors/slow/baseline keep policies:
  ~5% of head-sampled traces survive tail (errors + slow + 1% baseline)
  -> 250 traces/sec * 5 spans = 1,250 spans/sec into Tempo
```

Two orders of magnitude reduction, and **errors are still preserved at 100%** because they survive both layers (head: random; tail: explicit error policy).

### The error-recovery problem

The single most consequential property of head sampling: **head-sampled-out traces are gone**. Tier-1 head sampling will randomly drop the 99% it didn't pick. If an error occurs in a dropped trace, you cannot recover it. Tail sampling cannot recover what was never emitted.

This is the reason the production combo above runs head sampling *generously* (10-25%, not 0.1%): you want enough traces flowing into the gateway that the tail sampler has a representative population to work with. If your head sampling is too aggressive, you starve the tail sampler of errors to detect.

The decision lever:

| Head sampling rate | Tail sampler effectiveness |
|---------------------|----------------------------|
| 100% (no head sampling) | Maximum -- catches every error. Maximum tail-buffer memory. |
| 25% | Excellent -- 25% of errors visible to tail; tail keeps all of them. 4x reduction in agent and gateway load. |
| 10% | Good -- 10% of errors visible to tail; statistical confidence on error rates per service. |
| 1% | Marginal -- only 1% of errors flow through; for very high-error-rate services this is enough to detect, for low-rate ones you'll miss intermittent issues. |
| 0.1% | Inadequate -- you'll miss most errors. |

The interview answer: **"Pick head sampling rate based on the highest-volume service's error budget. For a service with a 1-in-1000 error rate where each error costs $1000, you need >= 0.1% head sampling at minimum to see 1 error/day; in practice run 25%+ and let the tail sampler do the cost reduction."**

---

## Part 4: Span Processors and Exporters -- The SDK Plumbing

Yesterday's doc covered the Collector's processor pipeline. Inside the SDK there's a separate, parallel concept: the **span processor** -- the in-process pipeline between span end and OTLP egress. Two flavors:

### `BatchSpanProcessor` -- the production answer

The SDK's `BatchSpanProcessor` (BSP) is a background goroutine/thread/task that:

1. Accepts ended spans and pushes them onto an in-memory queue (`max_queue_size`, default 2048).
2. Periodically (`schedule_delay_millis`, default 5000) -- or when the queue hits `max_export_batch_size` (default 512) -- flushes the queue to the configured exporter via OTLP.
3. Does *not* block the application thread on export. The end-of-span call returns immediately; export is async.

Configuration:

```bash
OTEL_BSP_MAX_QUEUE_SIZE=2048
OTEL_BSP_MAX_EXPORT_BATCH_SIZE=512
OTEL_BSP_SCHEDULE_DELAY=5000
OTEL_BSP_EXPORT_TIMEOUT=30000
```

What this buys: end-of-span is essentially free (queue push). What this costs: spans are lost if the process crashes between queue push and OTLP flush. This is the "BatchSpanProcessor lost spans on crash" gotcha -- a Pod OOM-kill can lose up to 5 seconds of trace data (or up to 2048 spans) per pod.

The mitigation: **the Collector's persistent queue, not the SDK's**. The SDK ships spans to the agent quickly; the agent's `sending_queue.storage: file_storage` extension persists to disk and survives Collector restarts. SDK persistence is rarely added; the architecture is "SDK fast-flushes; Collector persists."

### `SimpleSpanProcessor` -- never in production

The SimpleSpanProcessor (SSP) blocks the calling thread on every span end:

```
span.end() -> SSP.on_end(span) -> exporter.export([span]) -> blocks until OTLP returns
```

What this means in practice: every span end becomes a synchronous gRPC call to the Collector. If the Collector is slow or the network is congested, your application thread sits in the export call.

The use cases that justify SSP:

- **Unit tests** -- you want spans to be flushed and verifiable immediately after the operation under test.
- **Short-lived CLI tools / batch jobs** -- the process exits right after the operation, BSP's background flush never gets to run before exit.
- **Lambda / FaaS** -- the function execution ends and the runtime freezes the process before BSP would have flushed. (Even here, the AWS Lambda OTel layer uses an in-process exporter with a forced flush hook, not naked SSP.)

In a long-running production service, **SSP is a production-disaster waiting to happen.** Symptoms: tail latency spikes that correlate exactly with Collector restarts. p99 of every endpoint suddenly jumps 50ms when the gateway has a brief CPU spike.

The SDK config to make this hard to do by accident:

```python
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

provider.add_span_processor(
    BatchSpanProcessor(  # ALWAYS Batch in prod, never Simple
        OTLPSpanExporter(endpoint="http://otel-agent.observability.svc:4317"),
        max_queue_size=2048,
        max_export_batch_size=512,
        schedule_delay_millis=5000,
    )
)
```

### Exporter back-pressure and the SDK queue overflow

When the agent is unreachable, the SDK's BSP queue fills up. Default behavior: **drop newest spans when queue is full** (the SDK's "drop on full queue" is recoverable; the alternative -- block on enqueue -- would be a worse failure mode because it'd add latency to the application's request path). The dropped-span counter is exposed as the SDK's own metric (`otel_sdk_spans_dropped_total` or equivalent per-language), which you should scrape and alert on.

The retry semantics are part of the OTLP exporter, not the BSP:

- **gRPC**: `UNAVAILABLE`, `DEADLINE_EXCEEDED`, and a few others trigger retry with exponential backoff.
- **`INVALID_ARGUMENT`**: not retried -- the batch is malformed and would never succeed.
- **HTTP**: 5xx retry; 4xx don't retry.

Yesterday's gotcha #9 reinforced the asymmetry: the Collector is the one that 400s a malformed batch; the SDK reports success because the gRPC call returned *something*. Watch the *Collector's* exporter-failed counters, not just the SDK side.

### Why SDK-direct-export is fragile

Yesterday's gotcha #4 said: never SDK-direct in production. Today, with the trace-side detail, the reasoning is sharper:

| Failure mode | SDK direct to backend | SDK -> Collector -> backend |
|--------------|------------------------|------------------------------|
| Backend slow / down | SDK queue fills, drops spans, app gets slow if SSP | Collector queue absorbs; agent pods retry; persistent queue survives long outages |
| Per-pod connection storm at restart | 1000 pods open 1000 connections to backend at once | 1000 pods open 1000 connections to *agent* (cheap, local); agents fan into 5 gateway connections; gateway holds N connections to backend |
| Cross-cutting policy (sampling, redaction, multi-tenancy) | Implemented in 200 microservice configs | One Collector config, ops-owned |
| Vendor switch | Code change in every service | Collector exporter config change |
| Network partition | Spans lost (BSP queue size cap) | Persistent queue + retry survives |

The rule: **always `SDK -> agent (DaemonSet) -> gateway (Deployment) -> backend`**. The agent and gateway are not optional production components; they are the *production* part.

---

## Part 5: Storage Backends -- Tempo vs Jaeger vs X-Ray vs Vendor

The trace backend stores recorded spans and serves queries. The choice has order-of-magnitude cost and operational implications.

### Grafana Tempo -- object-storage-backed columnar blocks

Tempo's design innovation (now 4 years old; mainstream in 2026): **trace storage is object-storage-only.** No Cassandra, no Elasticsearch, no Postgres. S3 / GCS / Azure Blob / MinIO is the entire persistence layer. The architecture:

```
[Distributors] -- accept OTLP, hash by trace_id, forward to ingesters
       |
       v
[Ingesters]    -- buffer in memory, flush to object storage as Parquet blocks
       |
       v
[S3 / GCS]     -- compressed columnar Parquet, partitioned by tenant + time window
       ^
       |
[Queriers]     -- read blocks from S3, evaluate TraceQL, return results
       ^
       |
[Query frontend] -- shards queries across queriers
       ^
       |
       Grafana
```

Properties:

- **Cheap storage**: S3 at $0.023/GB/month for typical retention (30-90 days) makes Tempo dramatically cheaper than Cassandra/ES-backed alternatives. A petabyte of trace history costs ~$23k/month in S3, compared to ~$200k-$500k for an Elasticsearch cluster of equivalent capacity.
- **No ingest-time indexing of attributes** by default. Tempo originally supported only *trace ID lookup*; full-text attribute search came with TraceQL via the bloom-filter and column-pruning approach -- queries scan blocks but skip irrelevant columns and bloom-filter past blocks with no matching values. This is much slower than ES's inverted index for cold queries but cheap-to-store; the tradeoff is query latency vs storage cost.
- **Metrics-generator subsystem** (Tempo 2.0+) emits service-graph metrics and span-derived RED metrics from the ingest stream, similar to the spanmetrics connector but built in. Versions 2.6+ support TraceQL Metrics for ad-hoc trace analytics ("compute p95 of `db.query` spans grouped by `db.system` over the last 6 hours").

Versioning as of early 2026: **Tempo 2.6+** is the recommended production line; the major recent additions are TraceQL Metrics (instant trace analytics without pre-aggregation) and improved structural query performance.

### Jaeger -- the traditional model

Jaeger (CNCF graduated, contemporary with Zipkin's lineage) uses a different storage model:

- **Cassandra** (the original) or **Elasticsearch / OpenSearch** as the primary backend.
- Each span is indexed by trace ID, service name, operation name, and tag values.
- Searches are indexed lookups; very fast for simple queries.

Jaeger's strengths: cold-query latency (the index is always available), tag-based search (good for "find traces where `error=true`"), mature operations story for teams already running Cassandra.

Jaeger's weaknesses in 2026:

- **Storage cost**: Cassandra and ES are expensive at scale; index size often exceeds the actual span data.
- **Structural queries are weak**: Jaeger can search by tags but cannot express "show me traces where the auth service called user service which then errored" -- the structural-query feature TraceQL was built for.
- **OTel ecosystem alignment**: Tempo is Grafana-native; Jaeger has OTel integration but Tempo is the "natural choice" in a Grafana stack.

Jaeger remains the right answer when:

- You already operate Cassandra/ES and don't want to add object storage as a dependency.
- You need sub-second cold-query latency on a large index.
- You're constrained to on-prem with no S3-compatible storage.

### AWS X-Ray -- managed, with sampling rules

X-Ray is AWS's in-house trace backend. Architecture:

- **X-Ray Daemon** (the legacy collector; replaced by ADOT for OTel-native flows) accepts spans and forwards to the X-Ray service.
- **Sampling rules**: X-Ray has a centralized sampling rules engine -- you configure rules in the X-Ray service (e.g. "sample 10% of `/api/checkout` traces, 100% of error traces, 1 trace/sec/host minimum") and the rules are pulled by the daemon/SDK and applied at ingest time. This is *head sampling with central control*, distinct from OTel's per-SDK config.
- **Service Map** is built-in and reconstructed from CLIENT/SERVER span pairs.
- **Storage**: managed by AWS; you don't see the underlying store.
- **Cost model**: per-trace-recorded + per-trace-retrieved + retention. The sampling rules engine is what makes the cost predictable.

The OTel path to X-Ray:

1. Use the **ADOT (AWS Distro for OpenTelemetry) Collector** which bundles the `awsxray` exporter.
2. The exporter translates OTLP to X-Ray's wire format and uses the X-Ray service's sampling rules.
3. ADOT is itself just OTel Collector contrib + AWS-curated component selection + IRSA-friendly defaults.

When X-Ray makes sense:

- AWS-native shop, no desire to operate Tempo or pay a vendor APM.
- AWS-managed backend simplicity wanted (single bill, IAM-controlled access).
- Service Map and AWS service integration (API Gateway, ALB, Lambda all emit to X-Ray natively).

When it doesn't:

- Multi-cloud or hybrid deployments (X-Ray is AWS-only).
- Need deep structural query (X-Ray's query language is much weaker than TraceQL).
- Tight integration with non-AWS observability tooling.

### Datadog APM, Honeycomb, New Relic -- vendor APM

The vendor APM tier in 2026:

| Vendor | Strengths | When to pick |
|--------|-----------|--------------|
| **Datadog APM** | Polished UI, deep code-level profiling integration, strong AI-assisted RCA, extensive language SDK coverage, native OTLP ingest | Existing Datadog shop; want one vendor for metrics+logs+traces+RUM |
| **Honeycomb** | Best-in-class high-cardinality column store; industry-leading on "drill into one slow request and slice by 12 dimensions"; Refinery for tail sampling | Engineering-led teams that want analytic-database-style trace queries; SLO-driven org |
| **New Relic** | Strong service map, free tier with 100GB ingest, pay-per-user pricing | Shops standardized on NR for years; wide tooling already in place |
| **Splunk Observability** | Strong SignalFx metrics integration, AI Director root-cause analysis | Splunk-shop |
| **Lightstep / Cloud Observability** (now ServiceNow) | Change intelligence, deviation detection | Shops that want change-context overlaid on traces |
| **Dynatrace** | OneAgent end-to-end auto-instrumentation, Davis AI | Enterprise shops valuing single-agent simplicity |

The 2026 commonality: **all of them ingest OTLP natively**, so the *instrumentation* decision is decoupled from the *backend* decision. You can run OTel SDKs everywhere and switch the Collector's exporter from `otlphttp/datadog` to `otlphttp/honeycomb` without touching application code.

### The decision framework

| Situation | Pick |
|-----------|------|
| Grafana stack, K8s-native, want cheap retention on object storage | **Tempo** |
| Existing Cassandra/ES operations team, no S3 dependency wanted | **Jaeger** |
| AWS-only fleet, want managed simplicity, fine with weaker query | **X-Ray (via ADOT)** |
| Want the polish + RCA + RUM stitching, willing to pay | **Datadog or Dynatrace** |
| High-cardinality slicing as the primary use case, SLO-driven org | **Honeycomb** |
| Multi-vendor / no-lock-in transition | **Tempo as primary + vendor as secondary** (fan-out via Collector) |

---

## Part 6: TraceQL -- The Investigators' Query Language

TraceQL is to Tempo what PromQL (Apr 21 deep dive) is to Prometheus. Grafana Labs deliberately modeled the syntax to feel familiar. The language has three layers: **span selectors**, **structural operators**, and **aggregates**.

### Span selectors -- match individual spans

The basic building block. Selectors match spans whose attributes/intrinsics satisfy predicates:

```traceql
{ .http.status_code = 500 }
```

The `{ ... }` is a span selector. The `.` prefix marks an *attribute*. Inside the braces, normal comparison operators (`=`, `!=`, `<`, `<=`, `>`, `>=`, `=~` regex, `!~` regex-not), boolean `&&` / `||`, and parenthesization work as expected:

```traceql
{ .http.status_code >= 500 && .http.method = "POST" }
{ .db.system = "postgresql" && .db.statement =~ "SELECT.*" }
{ resource.service.name = "checkout" }
{ name = "GET /api/users/:id" }
```

**Attributes vs intrinsics vs resource attributes** -- the `.` vs `:` mental shortcut: **attributes use `.`, intrinsics use `:`**.

| Reference | Where it lives |
|-----------|----------------|
| `.attribute_name` | Span attribute (set via `span.set_attribute`) |
| `resource.attribute_name` | Resource attribute (the `service.name`, `k8s.pod.*` set on the SDK Resource) |
| `span:name`, `span:kind`, `span:status`, `span:duration` | Span intrinsics |
| `span:id`, `span:parentID`, `span:childCount`, `span:statusMessage` | More span intrinsics |
| `trace:rootName`, `trace:rootService`, `trace:duration` | Trace-level intrinsics (the root span's name and the root span's `service.name`) |
| `event:name`, `event:timeSinceStart` | Span-event intrinsics (the timestamped sub-points within a span) |
| `link:traceID`, `link:spanID` | Span-link intrinsics |
| `instrumentation:name`, `instrumentation:version` | Which instrumentation library produced the span |

Intrinsics worth memorizing:

```traceql
{ span:name = "POST /checkout" }
{ span:kind = server }                      # values: server, client, producer, consumer, internal
{ span:status = error }                     # values: error, ok, unset
{ span:duration > 500ms }                   # supports ns/us/ms/s/m/h
{ trace:rootName = "GET /api/checkout" }    # filter by root-span name (Tempo 2.4+)
{ trace:rootService = "ingress-nginx" }     # filter by root span's service.name
```

The bare unscoped form (`name = ...` without the `span:` prefix) was supported in early Tempo versions and still parses, but the prefixed form is canonical in Tempo 2.6+ and what you should use in any query you save to a dashboard or alert rule. Mixed-form queries are valid but harder to review.

### Structural operators -- the killer feature

This is where TraceQL diverges from anything Jaeger could express. Structural operators describe relationships *between spans within a trace*. The full catalog comes in three families:

**Logical operators** (combine filters within or across spansets, no structural meaning):

| Operator | Meaning |
|----------|---------|
| `&&` | Both filters must match (logical AND -- can be within a single span: `{ .a = 1 && .b = 2 }`) |
| `\|\|` | Either filter matches (logical OR) |

**Descendant / ancestor / sibling structural operators** (return only the *right-hand* side of the match):

| Operator | Meaning |
|----------|---------|
| `>>` | Descendant -- right-hand span is anywhere below left-hand span in the trace tree |
| `<<` | Ancestor -- right-hand span is anywhere above left-hand span (the inverse of `>>`) |
| `>` | Direct child -- right-hand span is the immediate child of left-hand span |
| `<` | Direct parent -- right-hand span is the immediate parent |
| `~` | Sibling -- right-hand span shares the same parent as left-hand span |

**Union structural operators** (return matches from *both* sides -- useful when you want to see both the auth span and the errored user span in your result, not just the user span):

| Operator | Meaning |
|----------|---------|
| `&>>` / `&<<` | Union descendant / union ancestor |
| `&>` / `&<` | Union direct-child / union direct-parent |
| `&~` | Union sibling |

**Not-relation operators** (negated structural relationships):

| Operator | Meaning |
|----------|---------|
| `!>>` / `!<<` | NOT-descendant / NOT-ancestor |
| `!>` / `!<` | NOT-direct-child / NOT-direct-parent |
| `!~` | NOT-sibling |

The mental shortcut: **bare `>>` returns the matched right-hand span; `&>>` returns both sides; `!>>` excludes traces where the relation holds**.

The killer query that motivates the whole language:

```traceql
{ resource.service.name = "auth" }
  >>
{ resource.service.name = "user" && span:status = error }
```

In English: *"Find traces where the auth service has a span which is the ancestor of a user-service span that errored."* In Jaeger's tag-based search, this is impossible to express without scanning every trace where auth and user co-occur and post-filtering.

More structural query examples:

```traceql
# Traces where checkout called payment, payment took > 1s, and payment errored
# Returns just the matching payment spans
{ resource.service.name = "checkout" }
  >> { resource.service.name = "payment" && span:duration > 1s && span:status = error }

# Same query but return BOTH the checkout and payment spans (union variant)
{ resource.service.name = "checkout" }
  &>> { resource.service.name = "payment" && span:status = error }

# Traces where a DB query was a *direct child* of an auth span (i.e. auth queried the DB,
# not "auth eventually led to a query 5 hops down")
{ resource.service.name = "auth" }
  > { .db.system = "postgresql" }

# Sibling DB queries -- two postgres queries with the same parent span
# (often a sign of N+1 query antipattern)
{ .db.system = "postgresql" }
  ~ { .db.system = "postgresql" }

# Traces where checkout was called and at no point did fraud-check run
# (use case: fraud check should always run, alert if it didn't)
{ resource.service.name = "checkout" }
  !>> { resource.service.name = "fraud-check" }
```

### Aggregates -- counting and reducing per trace

TraceQL supports aggregates with the `|` pipe (PromQL fans should feel at home):

```traceql
# Count traces matching the selector
{ resource.service.name = "checkout" && span:status = error }
  | count() > 5

# Average duration of payment spans, only return traces where avg > 800ms
{ resource.service.name = "payment" }
  | avg(span:duration) > 800ms

# Maximum status code attribute value across spans in the trace
{ resource.service.name = "checkout" }
  | max(.http.status_code) >= 500

# Sum of bytes over all DB queries in the trace
{ .db.system = "postgresql" }
  | sum(.db.rows_returned) > 10000
```

The aggregate runs **per trace** -- a trace passes the filter if its aggregate over matching spans satisfies the comparison.

### TraceQL Metrics -- the 2.6+ time-series feature

Tempo 2.6+ adds **TraceQL Metrics**: aggregations not over individual traces but over the entire trace stream as a time series. This is the feature that closes the analytics gap with Honeycomb and makes Tempo a viable answer to "I want PromQL-style queries directly against my trace data, not just pre-aggregated spanmetrics."

The functions, mapped from PromQL where the analogue is direct:

| TraceQL Metrics function | PromQL analogue | What it does |
|--------------------------|-----------------|--------------|
| `rate()` | `rate()` | Rate of matching spans per second |
| `count_over_time()` | `count_over_time()` | Count of matching spans in the time window |
| `quantile_over_time()` | `histogram_quantile()` | A percentile of a numeric attribute over time |
| `avg_over_time()` | `avg_over_time()` | Average of a numeric attribute |
| `min_over_time()` / `max_over_time()` | `min_over_time()` / `max_over_time()` | Min / max |
| `sum_over_time()` | `sum_over_time()` | Sum of a numeric attribute |
| `histogram_over_time()` | (no direct analogue -- it returns a heatmap) | Distribution of a numeric attribute as a histogram per time bucket; renders as a heatmap in Grafana |
| `compare()` | (no PromQL analogue) | Tempo-specific: compares two filter sets and shows which attributes differ -- the "what's different about errored traces vs successful ones" query |

Worked examples:

```traceql
# Rate of error spans per service over time -- the time-series equivalent of
# the per-trace error count we showed earlier
{ span:status = error } | rate() by (resource.service.name)

# p95 duration of checkout spans, as a time series -- without ever defining a histogram
# (compare to PromQL: histogram_quantile(0.95, sum by (le) (rate(spanmetrics_duration_seconds_bucket[5m]))))
{ resource.service.name = "checkout" } | quantile_over_time(span:duration, .95) by (resource.service.name)

# Heatmap of DB query duration over time, broken down by db.system
{ .db.system != nil } | histogram_over_time(span:duration) by (.db.system)

# Average rows-returned by DB statement type -- a query that would require
# custom instrumentation under spanmetrics, free here
{ .db.system = "postgresql" } | avg_over_time(.db.rows_returned) by (.db.operation)

# The "what's different" query: among checkout traces, how do error traces
# differ from successful ones?
{ resource.service.name = "checkout" } | compare({ span:status = error })
```

**Cost vs spanmetrics tradeoff** -- the decision framework:

| | Spanmetrics (PromQL) | TraceQL Metrics |
|---|----------------------|-----------------|
| Pre-aggregation | At ingest, in the connector | None -- full-block scan at query time |
| Cardinality control | At ingest via `dimensions` config | At query time -- queries can be expensive |
| Latency | Sub-second (Mimir-cached) | Seconds-to-minutes for long windows over large blocks |
| Flexibility | Limited to dimensions you decided to keep at ingest | Any attribute, any aggregation, ad-hoc |
| Time range | Bounded by Mimir retention (typically days) | Bounded by Tempo retention (typically weeks-months) |
| Use case | Dashboards, alerts | Investigation, ad-hoc analysis, "what attribute is different about these failing traces?" |

The pragmatic answer: **emit spanmetrics for dashboards and alerts; reach for TraceQL Metrics for investigation and ad-hoc questions.** Tempo's metrics-generator subsystem (see Tempo's own observability stack) actually runs a built-in spanmetrics + service-graph generator that emits to Prometheus, so you can keep both layers in the same backend.

### TraceQL vs PromQL -- the side-by-side

| Aspect | PromQL (Apr 21) | TraceQL |
|--------|-----------------|---------|
| Domain | Time series of metrics | Traces and spans |
| Selector syntax | `{ label = "value" }` | `{ .attribute = "value" }` for span attrs; `{ resource.x = ... }` for resource attrs; `{ span:name = ... }` for intrinsics |
| Sigil convention | None -- everything is a label | `.` = attribute, `:` = intrinsic. Memorize this. |
| Time semantics | Range vector `[5m]`, instant vector | Per-trace evaluation by default; TraceQL Metrics adds time-series |
| Joins | `on (label) group_left` | Structural operators `>>`, `>`, `~`, `&>>`, `!>>` -- joining is by *trace tree* not label set |
| Aggregates | `sum`, `avg`, `count` etc. with `by`/`without` | `count()`, `avg()`, `max()`, `sum()` per trace; `rate()`, `quantile_over_time()`, `histogram_over_time()`, `compare()` for TraceQL Metrics |
| Mental model | "How is this number changing over time?" | "Which traces match this structural pattern, and how many?" |

The mental shortcut: **PromQL queries the FAA monthly statistics; TraceQL queries the individual flight recordings**. They answer different questions; you'll use both.

---

## Part 7: Trace <-> Metrics <-> Logs Correlation -- Closing the Triangle

The whole point of OTel's three-signal unification is that you can pivot from any signal to any other. Three correlation links matter; two were introduced earlier in the journal, and the spanmetrics connector is the one we go deep on today.

### Link 1: traces -> logs (via `trace_id` injection)

A log line emitted inside a span should carry the span's `trace_id`. Yesterday's Part 1 covered this in the "three-layer trace-to-logs correlation" model:

1. The application logger must be wired to OTel context so it picks up the active span's trace ID.
2. The log record must expose `trace_id` as a queryable field (top-level OTLP field, JSON top-level, Loki structured metadata, or derived field).
3. Grafana's data source must know how to construct the click-through query.

The pivot: in Grafana, click a span in Tempo's trace waterfall -> "View related logs" -> Loki query with `| trace_id = "<the_id>"`.

#### The 2am diagnostic when "View related logs" returns empty

This is one of the most common production incidents in observability rollouts and deserves an explicit failure-mode catalog. **The 30-second triage that splits the entire problem space**: open one of the logs you *can* find with a manual query and look for the trace_id literally in the log line text. If trace_id is absent, the cause is upstream of Grafana (instrumentation gap); if present, the cause is in the click-through query (format, time window, or label filter mismatch).

Five distinct failure modes produce the symptom "View related logs returns nothing":

1. **Trace_id absent from log line entirely** -- the most common cause. The OTel SDK does NOT auto-inject trace context into stdlib loggers. You need an explicit logging integration per language: Java MDC bridge (`opentelemetry-logback-mdc-1.0`), Python `LoggingInstrumentor` (autopatches `logging`), Go `slog` handler with OTel context propagation, .NET `ActivityListener` (auto-enriches `ILogger`). Without this, the span is recorded but the log line carries no trace_id and no substring match in Loki will ever find it.

2. **Format / case mismatch** -- the log emits `traceId=ABC-123-DEF` (UUID-hyphenated, uppercase) while Tempo and Grafana use 32-char lowercase hex. Common when a logging library predates OTel adoption or a custom JSON formatter normalizes IDs. The substring search is literally not finding the right text. Fix: align the log format to the OTel canonical 32-char lowercase hex emitted by `span.context.trace_id`.

3. **Time window too narrow** -- Trace to Logs defaults to ±5min around the span (`spanStartTimeShift: -1h`, `spanEndTimeShift: 1h` in modern data source configs override this). Logs emitted outside the window from long-running async work, retry storms, or clock-skewed hosts get missed. Fix: widen the shifts in the data source, or use a manual exploratory query during incidents.

4. **Label filter template mismatch** -- the data source's Trace to Logs config has a hard-coded or templated label filter (`{cluster="$cluster"}`, `{namespace="$namespace"}`) that doesn't intersect with where the logs actually live. Common with multi-cluster setups where the variable interpolation fails. Fix: use forwarded trace tags for the label filter (`tags: [{key: 'service.name', value: 'service_name'}]` so the filter auto-matches the calling service's stream).

5. **Loki `derivedFields` regex not configured** -- the trace_id is in the log line as plain text, but Loki's data source isn't configured with the `derivedFields` regex extractor to surface it as a structured-metadata field with a click-through link. The substring `|=` match works but the structured trace_id pivot doesn't. Fix: add to the Loki data source's `jsonData.derivedFields`:

```yaml
derivedFields:
  - name: trace_id
    matcherRegex: 'trace_id=(\w+)'    # or "traceID":"(\w+)" for JSON logs
    url: '${__value.raw}'
    datasourceUid: tempo
```

The architectural truth that closes the loop: **logs and traces don't auto-correlate.** Tracing instrumentation gives you spans but does not stamp trace_ids into log records unless you separately wire the logging library into the OTel context. The "View related logs" button is a dumb LogQL substring search whose success depends entirely on the discipline of upstream instrumentation. Once the linkage is correct, every trace becomes a one-click pivot into its full log narrative.

### Link 2: traces -> metrics (via spanmetrics connector)

This is the unlock for "click a metric latency spike, jump to a representative trace." Two halves:

**Half A: spanmetrics connector emits derived metrics from spans.**

Yesterday's Part 4 introduced the connector concept: an exporter for one pipeline that's also a receiver for another. The `spanmetrics` connector is the canonical example:

```yaml
connectors:
  spanmetrics:
    namespace: spanmetrics                    # metric name prefix
    histogram:
      explicit:
        buckets: [10ms, 25ms, 50ms, 100ms, 250ms, 500ms, 1s, 2.5s, 5s]
    dimensions:                               # span attributes -> metric labels
      - name: http.method
      - name: http.status_code
      - name: http.route                      # bounded; route templates not raw URLs
    exemplars:
      enabled: true                           # ATTACH trace_id exemplars to histograms
    metrics_flush_interval: 30s
    aggregation_temporality: "AGGREGATION_TEMPORALITY_CUMULATIVE"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, tail_sampling, batch]
      exporters: [otlp/tempo, spanmetrics]    # spans go to Tempo AND into the connector
    metrics/from-spans:
      receivers: [spanmetrics]                # connector emits metrics into THIS pipeline
      processors: [memory_limiter, batch]
      exporters: [prometheusremotewrite/mimir]
```

What the connector emits per `(service.name, span.name, dimensions)` tuple:

```
spanmetrics_calls_total{service_name, span_name, http_method, http_status_code, http_route} -- counter
spanmetrics_duration_seconds_bucket{...,le="0.025"} -- histogram bucket count
spanmetrics_duration_seconds_count{...} -- histogram count
spanmetrics_duration_seconds_sum{...} -- histogram sum
```

These are RED metrics (Rate, Errors, Duration) automatically derived from the trace stream. PromQL queries that work immediately:

```promql
# Request rate by service
sum by (service_name) (rate(spanmetrics_calls_total[5m]))

# Error rate by service
# NOTE: the status_code label value is the proto enum name (STATUS_CODE_ERROR) in modern
# spanmetrics connector versions. Older versions (pre-0.80 contrib) emitted "Error" / "Ok" /
# "Unset" (camelCase). Match the label value to your connector version -- a mismatch here
# silently makes your error-rate query return zero, which looks like "we have no errors"
# until 3am.
sum by (service_name) (rate(spanmetrics_calls_total{status_code="STATUS_CODE_ERROR"}[5m]))
  /
sum by (service_name) (rate(spanmetrics_calls_total[5m]))

# p95 duration by service
histogram_quantile(0.95,
  sum by (service_name, le) (rate(spanmetrics_duration_seconds_bucket[5m]))
)
```

This is the **RED method on autopilot**: you wrote zero metric instrumentation; the spans you were already emitting produced the metrics.

The `dimensions:` config is where teams blow it up. **Every dimension becomes a metric label.** Adding `user.id` or `http.url` (with raw URLs including IDs) is the canonical cardinality bomb -- a single misconfig has taken down Mimir clusters. The discipline:

```yaml
# GOOD dimensions (bounded cardinality)
dimensions:
  - name: http.method                      # ~10 values
  - name: http.status_code                 # ~50 values
  - name: http.route                       # bounded by service's route table (~50-200)
  - name: db.system                        # 5-10 values
  - name: messaging.destination_kind       # 3 values

# BAD dimensions (unbounded cardinality)
dimensions:
  - name: user.id                          # MILLIONS of values per service
  - name: http.target                      # raw URLs with IDs -> millions of values
  - name: db.statement                     # SQL with literal values -> millions
  - name: trace_id                         # NEVER -- one series per trace
```

The default dimensions emitted with no config are `service.name`, `span.name`, `span.kind`, `status.code` -- all bounded. Add only attributes you've audited.

**Half B: exemplars attach `trace_id` to specific histogram observations.**

The `exemplars.enabled: true` flag is what closes the loop. With exemplars enabled, the spanmetrics connector attaches the originating `trace_id` to histogram bucket observations. The Apr 20 Prometheus doc and Apr 24 Grafana doc both covered exemplars -- a single observation in a histogram bucket carries a representative `trace_id` pointer.

In Grafana:

1. User views a panel of `histogram_quantile(0.99, ...)` over `spanmetrics_duration_seconds_bucket`.
2. Grafana renders the latency line, with little "diamond" markers for exemplars.
3. User clicks a slow-bucket exemplar diamond.
4. Grafana opens Tempo with the exemplar's `trace_id` -> waterfall of the actual slow trace.
5. User clicks a slow span -> "View related logs" -> Loki shows that span's logs.

Three signals, three click-throughs, fully connected.

### Link 3: metrics -> traces (via exemplars, in reverse)

Same exemplar mechanism, opposite click direction. The Apr 24 Grafana doc covered this Tempo-data-source configuration: a Tempo data source can be linked to a Mimir data source so that exemplar markers from Prometheus histograms (regardless of whether they were emitted by spanmetrics or by manual instrumentation in the application) act as click-throughs into Tempo trace lookups by trace ID.

The key Grafana data source config (the bits that matter):

```yaml
# Grafana provisioning datasources/tempo.yaml
apiVersion: 1
datasources:
  - name: Tempo
    type: tempo
    uid: tempo
    url: http://tempo-query-frontend.observability.svc:3200
    jsonData:
      tracesToLogsV2:
        datasourceUid: loki
        tags: [{ key: 'service.name', value: 'service_name' }]
        spanStartTimeShift: -1h
        spanEndTimeShift: 1h
      tracesToMetrics:
        datasourceUid: mimir
        tags: [{ key: 'service.name', value: 'service_name' }]
      serviceMap:
        datasourceUid: mimir
      nodeGraph:
        enabled: true
      lokiSearch:
        datasourceUid: loki
```

And on the Mimir data source side, exemplars must be enabled:

```yaml
- name: Mimir
  type: prometheus
  url: http://mimir-query-frontend.observability.svc/prometheus
  jsonData:
    exemplarTraceIdDestinations:
      - name: trace_id
        datasourceUid: tempo
        urlDisplayLabel: View Trace
```

Both sides referenced by `uid` so the click-through works in both directions.

### Service graph -- another side effect of span.kind

The Tempo metrics-generator subsystem (and the standalone `servicegraph` connector for non-Tempo backends) emits service-to-service edge metrics:

```
traces_service_graph_request_total{client="checkout", server="auth"}
traces_service_graph_request_failed_total{client="checkout", server="auth"}
traces_service_graph_request_duration_seconds_bucket{client="checkout", server="auth", le="..."}
```

These are reconstructed from CLIENT/SERVER span pairs across services. Grafana's "Service Graph" panel renders the full cluster's service topology from this metric set. It is dynamically rebuilt -- if a new service starts emitting CLIENT spans into auth, the edge appears in the graph automatically.

This is why span.kind matters operationally: **the service graph is wrong silently when kinds are wrong.** A SERVER span tagged INTERNAL means the receiver isn't drawn as a node. A CLIENT span tagged INTERNAL means the edge isn't drawn. Auto-instrumentation gets this right; manual spans are where fleets break it.

#### The killer insight: pairing, not tree-walking

A common misconception is that the service graph is built by walking the trace tree (parent_span_id chains). It is **not**. The mechanism is **stream pairing**: the connector buffers CLIENT spans, waits for a matching SERVER span to arrive within a configurable `wait` window (default ~10s), and emits an edge metric when the pair completes. Unmatched halves are tracked via `traces_service_graph_unpaired_spans_total` and dropped after expiry via `traces_service_graph_expired_edges_total`.

This pairing-based architecture has a **critical implication: the service graph connector has the same architectural constraint as the `tail_sampling` processor** -- both halves of a relationship must arrive at the *same* connector replica or the pair is never matched. Run the metrics-generator (or `servicegraph` Collector connector) behind the same loadbalancing tier with `routing_key: traceID` consistent hashing that fronts your tail-sampling pool. **One architectural pattern solves two stateful trace-processing problems** -- the front gateway running `loadbalancing` exporter into a back tier StatefulSet handling both tail sampling and service-graph generation, all sharded by trace_id.

Diagnostic decision tree when service-graph edges are missing despite traces flowing:

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `traces_service_graph_unpaired_spans_total` is steady-high | CLIENT and SERVER spans landing on different connector replicas | Front the connector with `loadbalancing` exporter, trace_id consistent hashing |
| `traces_service_graph_unpaired_spans_total` spikes around latency anomalies | `wait` window too short for slow trace duration | Raise `wait` to cover p99 trace duration, budget extra buffer memory |
| Specific edges missing but others present | `span.kind = INTERNAL` on what should be CLIENT or SERVER | Audit instrumentation; OTTL `transform` processor can correct kind as a stopgap: `set(span.kind, SPAN_KIND_CLIENT) where attributes["http.method"] != nil` |
| All edges missing for a specific service | Service emitting no CLIENT or SERVER spans | TraceQL diagnostic: `{ resource.service.name = "frontend" && span:kind = client } \| count() = 0` confirms |

---

## Part 8: Production Deployment Patterns -- The Tail-Sampling Topology Concretely

Yesterday introduced the agent + gateway pattern at a conceptual level. Today's contribution: the *concrete YAML* for a two-tier Collector pipeline doing tail sampling at scale.

### The two-tier topology

```
                        +-------------------------+
                        |  Apps (SDK 100% sample) |
                        +-----------+-------------+
                                    |
                                    | OTLP localhost:4317
                                    |
            +---------------------------------------------------+
            |        AGENT DAEMONSET (one per node)              |
            |        - k8sattributes, resource enrichment        |
            |        - filelog for container logs                |
            |        - kubeletstats for node metrics             |
            |        - batch                                     |
            |        - exporter: otlp -> front gateway           |
            +-------------------------+--------------------------+
                                      |
                                      | OTLP
                                      v
            +---------------------------------------------------+
            |   FRONT GATEWAY (Deployment, replicas=N, HPA)     |
            |   - receives OTLP                                  |
            |   - `loadbalancing` exporter (consistent hash by   |
            |     trace_id) routes to a downstream pool          |
            |   - NO sampling decision here                      |
            +-------------------------+--------------------------+
                                      |
                                      |  consistent-hash-by-trace_id
                                      |
            +-----------+-------------+--------------+----------+
            v           v             v              v          v
        +-------+   +-------+     +-------+      +-------+   +-------+
        | TAIL  |   | TAIL  | ... | TAIL  |      | TAIL  |   | TAIL  |
        | rep0  |   | rep1  |     | repN  |      | repN+1|   | repM  |
        +---+---+   +---+---+     +---+---+      +---+---+   +---+---+
            |           |             |              |           |
            |           |             |              |           |
            +-----------+-------------+--------------+-----------+
                                      |
                                      | OTLP
                                      v
                                  +-------+
                                  | TEMPO |
                                  +-------+
```

The front gateway is **stateless** and exists only to do consistent-hash routing. The tail-sampling pool is **stateful** (StatefulSet), holds the in-memory trace buffers, and is what runs the policy engine.

### Front gateway config (the loadbalancing layer)

```yaml
# front-gateway-values.yaml -- the loadbalancing tier
mode: deployment
replicaCount: 3
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 12
  targetCPUUtilizationPercentage: 70

image:
  repository: otel/opentelemetry-collector-contrib
  tag: 0.110.0                              # OTel Collector contrib 0.110+ as of early 2026

config:
  receivers:
    otlp:
      protocols:
        grpc: { endpoint: 0.0.0.0:4317 }
        http: { endpoint: 0.0.0.0:4318 }

  processors:
    memory_limiter:
      check_interval: 1s
      limit_percentage: 80
      spike_limit_percentage: 25
    batch:
      timeout: 1s                          # short -- minimize latency before tail layer
      send_batch_size: 1024

  exporters:
    loadbalancing:
      protocol:
        otlp:
          tls: { insecure: true }
          retry_on_failure: { enabled: true }
          sending_queue: { enabled: true, queue_size: 5000 }
      resolver:
        dns:
          hostname: tail-sampler-headless.observability.svc.cluster.local
          port: "4317"
          interval: 5s                     # re-resolve every 5s for pool changes
          timeout: 1s
      routing_key: traceID                 # hash by trace_id (default and only sane choice)

  service:
    telemetry:
      metrics:
        level: detailed
        address: 0.0.0.0:8888
    pipelines:
      traces:
        receivers: [otlp]
        processors: [memory_limiter, batch]
        exporters: [loadbalancing]
      metrics:
        receivers: [otlp]
        processors: [memory_limiter, batch]
        exporters: [otlp/mimir]            # metrics don't need sticky routing
      logs:
        receivers: [otlp]
        processors: [memory_limiter, batch]
        exporters: [otlp/loki]
```

Key points:

- **`loadbalancing` exporter routes by `traceID`** -- this is the bit that makes tail sampling work.
- **DNS resolver** points at the *headless* service of the tail-sampling StatefulSet. A headless Service returns all pod IPs (not the VIP), which is what the `loadbalancing` exporter consistently hashes against.
- **`interval: 5s`** for DNS re-resolution. Lower = faster reaction to pool changes, but more "split" windows where a trace's spans land on different replicas during scale events. 5s is a typical default; tighter intervals only matter at very high churn rates.
- **`batch` is short here** (1s, 1024) because we want spans flowing into the tail layer quickly so they're available for the decision_wait window.

### Tail-sampling pool config (the stateful layer)

```yaml
# tail-sampler-values.yaml
mode: statefulset                            # CRITICAL: stable pod identity for hash ring
replicaCount: 6                              # rule of thumb: 1 replica per ~5k spans/sec
podDisruptionBudget:
  enabled: true
  minAvailable: 4

image:
  repository: otel/opentelemetry-collector-contrib
  tag: 0.110.0

# Headless service so DNS returns pod IPs, not the VIP
service:
  type: ClusterIP
  clusterIP: None                            # makes it headless

config:
  receivers:
    otlp:
      protocols:
        grpc: { endpoint: 0.0.0.0:4317 }

  processors:
    memory_limiter:
      check_interval: 1s
      limit_percentage: 80
      spike_limit_percentage: 25

    tail_sampling:
      decision_wait: 30s                     # > p99.9 of slowest service in scope
      num_traces: 100000                     # max in-flight trace IDs per replica
      expected_new_traces_per_sec: 2000      # sizing hint
      decision_cache:
        sampled_cache_size: 100000
        non_sampled_cache_size: 100000

      policies:
        - name: errors-100pct
          type: status_code
          status_code: { status_codes: [ERROR] }

        - name: http-5xx-100pct
          type: numeric_attribute
          numeric_attribute:
            key: http.status_code
            min_value: 500
            max_value: 599

        - name: slow-100pct
          type: latency
          latency: { threshold_ms: 1000 }

        - name: healthcheck-1pct
          type: and
          and:
            and_sub_policy:
              - name: is-healthcheck
                type: string_attribute
                string_attribute:
                  key: http.route
                  values: [/healthz, /readyz, /metrics]
              - name: drop-most
                type: probabilistic
                probabilistic: { sampling_percentage: 1 }

        - name: vip-100pct
          type: string_attribute
          string_attribute:
            key: tenant.tier
            values: [enterprise, regulated]

        - name: baseline-1pct
          type: probabilistic
          probabilistic: { sampling_percentage: 1 }

    transform/sanitize:
      trace_statements:
        - context: span
          statements:
            - delete_key(attributes, "http.request.header.authorization")
            - delete_key(attributes, "http.request.header.cookie")

    batch:
      timeout: 5s
      send_batch_size: 8192

  connectors:
    spanmetrics:
      namespace: spanmetrics
      histogram:
        explicit:
          buckets: [10ms, 25ms, 50ms, 100ms, 250ms, 500ms, 1s, 2.5s, 5s]
      dimensions:
        - name: http.method
        - name: http.status_code
        - name: http.route
      exemplars:
        enabled: true                         # CRITICAL for click-through
      metrics_flush_interval: 30s

  exporters:
    otlp/tempo:
      endpoint: tempo-distributor.observability.svc:4317
      tls: { insecure: true }
      sending_queue:
        enabled: true
        storage: file_storage                 # persistent queue survives restarts
    prometheusremotewrite/mimir:
      endpoint: http://mimir-distributor.observability.svc/api/v1/push
      headers: { X-Scope-OrgID: prod-us-east }
      send_native_histograms: true            # cardinality win

  extensions:
    file_storage:
      directory: /var/lib/otel/queue
      timeout: 1s

  service:
    extensions: [file_storage]
    telemetry:
      metrics: { level: detailed, address: 0.0.0.0:8888 }
    pipelines:
      # Spans: tail-sample, sanitize, fan out to Tempo AND spanmetrics connector
      traces:
        receivers: [otlp]
        processors: [memory_limiter, tail_sampling, transform/sanitize, batch]
        exporters: [otlp/tempo, spanmetrics]

      # Metrics derived from spans: from connector to Mimir
      metrics/from-spans:
        receivers: [spanmetrics]
        processors: [memory_limiter, batch]
        exporters: [prometheusremotewrite/mimir]

resources:
  requests: { cpu: 1, memory: 2Gi }
  limits:   { cpu: 4, memory: 6Gi }            # tail buffer is memory-heavy

volumeClaimTemplates:                          # required for StatefulSet + file_storage
  - metadata: { name: queue }
    spec:
      accessModes: [ReadWriteOnce]
      resources: { requests: { storage: 10Gi } }
```

What this earns its keep on:

- **`mode: statefulset`** + **`clusterIP: None`** (headless): stable identity for the consistent-hash ring.
- **`tail_sampling` with the canonical policy stack**: errors + slow + tenancy overrides + healthcheck-suppression + 1% baseline.
- **`spanmetrics` connector with `exemplars: true`**: closes the metrics-to-traces click-through loop.
- **`file_storage` extension + `sending_queue.storage: file_storage`** on the Tempo exporter: persistent queue means a Collector restart doesn't lose buffered spans.
- **Spans fan out to Tempo AND spanmetrics in the same pipeline**: yesterday's "spans go out AND into the connector" pattern, concretely.
- **`send_native_histograms: true`** on the Mimir exporter: yesterday's gotcha #17 -- histograms emitted by spanmetrics drop ~10-50x in cardinality when shipped as native histograms vs classic buckets.

### Tempo via Helm (the backend)

```yaml
# tempo-values.yaml -- production Tempo on EKS via the upstream chart
# helm repo add grafana https://grafana.github.io/helm-charts
# helm install tempo grafana/tempo-distributed --version 1.18.0 -n observability -f tempo-values.yaml

storage:
  trace:
    backend: s3
    s3:
      endpoint: s3.us-east-1.amazonaws.com
      bucket: ${TEMPO_S3_BUCKET}
      region: us-east-1
      forcepathstyle: false
    pool:
      max_workers: 100
      queue_depth: 10000
    blocklist_poll: 5m

distributor:
  replicas: 3
  receivers:
    otlp:
      protocols:
        grpc: { endpoint: 0.0.0.0:4317 }
        http: { endpoint: 0.0.0.0:4318 }

ingester:
  replicas: 3
  persistence:
    enabled: true
    size: 100Gi
  config:
    max_block_duration: 30m
    trace_idle_period: 30s

querier:
  replicas: 3

queryFrontend:
  replicas: 2
  query:
    enabled: true                            # query frontend with Grafana data source

compactor:
  replicas: 1
  config:
    compaction:
      block_retention: 720h                  # 30 days
      compacted_block_retention: 1h

metricsGenerator:
  enabled: true                              # Tempo's built-in spanmetrics + service-graph
  config:
    processor:
      service_graphs:
        dimensions: [client, server]
      span_metrics:
        dimensions:
          - name: service.name
          - name: span.name
          - name: span.kind
          - name: status.code
    storage:
      remote_write:
        - url: http://mimir-distributor.observability.svc/api/v1/push
          send_exemplars: true               # required for the click-through

global_overrides:
  metrics_generator_processors: [service-graphs, span-metrics]
```

Two things worth noting on the Tempo side:

- **The metrics-generator subsystem is Tempo's built-in equivalent of the spanmetrics + servicegraph connectors.** You can run *both* (Collector connector + Tempo metrics-generator) but typically pick one to avoid double-counting. Pick the Collector connector if you want the metrics emitted close to the source (less network hop, fewer double-paths); pick Tempo's metrics-generator if you want fewer Collector components and accept that derived metrics live downstream.
- **`send_exemplars: true`** on metrics-generator's remote_write is what attaches `trace_id` exemplars to the histogram observations reaching Mimir. Without it, the Grafana click-through fails silently.

### Grafana data source for Tempo

Yesterday's Apr 27 doc and Apr 24 Grafana doc both touched on Tempo configuration; the relevant bit specifically for trace-correlation:

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
        datasourceUid: loki
        tags:
          - key: service.name
            value: service_name
        spanStartTimeShift: '-1h'
        spanEndTimeShift: '1h'
        filterByTraceID: true
      tracesToMetrics:
        datasourceUid: mimir
        tags: [{ key: 'service.name', value: 'service_name' }]
        queries:
          - name: 'Sample query'
            query: 'sum(rate(spanmetrics_calls_total{$$__tags}[5m]))'
      serviceMap:
        datasourceUid: mimir                  # service map driven by spanmetrics
      nodeGraph:
        enabled: true
      search:
        hide: false
      lokiSearch:
        datasourceUid: loki
      traceQuery:
        timeShiftEnabled: true
        spanStartTimeShift: '-1h'
        spanEndTimeShift: '1h'
      spanBar:
        type: 'Tag'
        tag: 'http.path'
```

The four boolean correlations in this config are what light up the click paths:

- `tracesToLogsV2` -> Tempo span -> Loki logs filtered by trace_id
- `tracesToMetrics` -> Tempo span -> Mimir metrics filtered by service tags
- `serviceMap` -> Tempo's service-graph metrics rendered as a topology
- Mimir's `exemplarTraceIdDestinations` (in the Mimir data source config) -> exemplar diamond click -> Tempo trace lookup

---

## Part 9: Production Gotchas (Pinned to the Wall)

A non-exhaustive list of the failure modes that bite distributed-tracing rollouts in production. Yesterday's gotchas covered the Collector layer; today's are trace-specific.

1. **Incomplete traces from sampling -- "I can find the request in the log but not in Tempo."** Head sampling dropped it. Mitigation: head sample generously (10%+); add a "force keep" attribute the app can set on suspect requests; tail-sample errors at 100%.

2. **Clock skew across services breaks duration math.** Span A on host 1 ends at `t=100ms`; span B (its child) on host 2 starts at `t=99ms` because host 2's clock is 5ms behind. The waterfall renders the child as starting *before* the parent's start. Worse: aggregate p95 latency calculations include negative durations that statistical libraries silently round to zero. Mitigation: deploy chrony/ntpd on every node; alert on `chrony_tracking_system_time_seconds` drift > 1ms; use `span.observed_timestamp` for ingest-side ordering when you need monotonic sequencing.

3. **Ingestion cost explosion from ungoverned span creation.** A bug in a manual instrumentation pattern (`for record in batch: tracer.start_span(...)`) starts producing 10,000 spans per request. Mitigation: dashboard `otelcol_receiver_accepted_spans` per service; alert on `> 3 sigma` deviation; quota at the gateway with `groupbyattrs` + `rate_limiting` per `service.name`.

4. **Span attribute cardinality bombs in spanmetrics `dimensions`.** Adding `user.id` or `http.url` to dimensions = a million-series Mimir cardinality bomb. Mitigation: audit the dimension list against the semantic-conventions list of *bounded* attributes; keep dimensions to `http.method`, `http.status_code`, `http.route` (templated, not raw URL), `db.system`, `messaging.destination_kind`. Never `user.id`, `trace_id`, `http.target`.

5. **Propagator misconfiguration drops traces at boundaries.** Service A is configured with `OTEL_PROPAGATORS=tracecontext`; service B is configured with `OTEL_PROPAGATORS=b3multi`. Service A sends `traceparent`; service B doesn't read it; trace fragments. Mitigation: standardize on `tracecontext,baggage` as the default; add `b3multi` and/or `xray` only for known legacy paths; verify with a synthetic test that emits a known trace_id and checks downstream span emission.

6. **`traceparent` mutated or stripped by ingress controllers.** NGINX Ingress, AWS ALB, older Istio/Envoy strip or rewrite headers. Trace's root span ends up being the *ingress*, not the app, and downstream apps create *new* root spans because they didn't see a parent. Mitigation: enable OpenTelemetry-aware NGINX module; for ALB use the `xray` propagator alongside `tracecontext`; for Istio 1.12+ propagation is native -- verify in test traffic.

7. **Missing context in async/queue handoffs requires manual propagation.** OTel auto-instrumentation for Kafka/SQS/RabbitMQ libraries usually injects `traceparent` into message headers automatically. But if you have a custom queue (Redis pub/sub, AWS Kinesis without a supported client, an internal RPC bus), the propagation has to be manual: `extract` on the consumer side, `inject` on the producer side. Mitigation: audit every queue/bus boundary in your architecture; write a propagation test; the bus library either supports it or you write the wrapper.

8. **`BatchSpanProcessor` lost spans on crash without persistent queue.** SDK BSP has an in-memory queue; on Pod OOMKill, the queue evaporates. Up to 5 seconds of trace data lost per pod restart. Mitigation: rely on the *Collector's* `file_storage` extension and `sending_queue.storage: file_storage` -- the SDK ships fast to the agent; the agent persists to disk. SDK persistence is rarely added because the agent path covers it.

9. **Head-sampled-out errors are unrecoverable forever.** If head sampling drops a trace at the SDK before any error is observed, no amount of tail sampling at the gateway can recover it. Mitigation: head sample generously (10-25%); accept the cost in exchange for tail-sampler error capture rate.

10. **Tail sampling `decision_wait` too short truncates slow traces.** Set `decision_wait: 30s` and a service whose p99.9 is 45s has its trace decision finalized *before* the slow span lands, fragmenting the trace. The decision was the right one (slow trace -> keep) but executed on incomplete data. Mitigation: set `decision_wait > p99.9_of_slowest_service_in_scope`; instrument `otelcol_processor_tail_sampling_late_span_age` to detect violations; accept the memory cost.

11. **Span.kind set wrong silently breaks the service map.** A SERVER span tagged INTERNAL means the receiver isn't drawn as a service-graph node. A CLIENT span tagged INTERNAL means the edge isn't drawn. Mitigation: rely on auto-instrumentation for HTTP/gRPC/DB calls (it gets kind right); for manual instrumentation, *always* set kind explicitly; verify the service map renders the edges you expect after every new manual instrumentation deploy.

12. **OTel SDK + Jaeger backend version skew.** Older Jaeger versions don't accept all OTLP fields (e.g. exponential histograms, span links with attributes, status messages over a certain length). Symptoms: silent drops at the Jaeger gRPC ingress, or "version mismatch" errors in Jaeger logs. Mitigation: deploy the OTel Collector's `jaeger` exporter (translates OTLP to Jaeger's wire format) rather than relying on Jaeger's native OTLP support, especially on Jaeger < 1.50.

13. **`tracestate` 32-entry overflow drops oldest entries first.** A request that crosses 32+ vendor-aware proxies (rare in practice but happens in heavy multi-vendor setups) overflows the cap. The drop is silent and deterministic (oldest goes first), which usually means *your own* tracestate entry is preserved while a downstream vendor's entry is dropped. Mitigation: limit `tracestate` writes to the small vendor list you actually need; audit propagator chain length.

14. **Tempo object-storage retention vs query cost.** Cheap storage (S3 at 30/60/90/365 days) does not equal cheap *queries*. A 90-day TraceQL search over 10 TB of blocks scans the entire range (unless trace_id-targeted). Mitigation: tune `compactor.compaction.block_retention` to the retention you actually use; use TraceQL's time-bounded `&& start_time > now() - 1h` qualifier where possible; cap the queryable retention separately from storage retention via Tempo's per-tenant overrides.

15. **Exemplars missing from histogram if spanmetrics connector is misordered in the pipeline.** If `tail_sampling` runs *after* `spanmetrics` (or in a different pipeline), the connector emits metrics from spans that were eventually sampled out. The exemplar `trace_id` then points at a trace that doesn't exist in Tempo -- click-through returns "trace not found." Mitigation: always run `spanmetrics` *after* `tail_sampling` in the traces pipeline so the connector sees only sampled-in spans; alternatively, run spanmetrics in a parallel pre-sampling pipeline if you want metrics from all traffic, but accept that ~95% of exemplars will then 404.

16. **`status = ok` filter misses most successful spans.** Auto-instrumentation sets status `Unset` on success; only sets `Ok` rarely. A TraceQL query `{ status = ok }` returns nearly empty. Mitigation: use `{ status != error }` for "successful" filtering. Document this gotcha in your team's TraceQL cheatsheet -- it bites every new user once.

17. **`loadbalancing` exporter DNS rebalance window opens a span-routing race.** During a tail-sampling pool scale-up, new pods enter the consistent-hash ring; the front gateway's DNS-resolve interval (default 5s) determines how quickly each replica learns about them. During the window, a fraction of in-flight traces have *later* spans routed to a different replica than their *earlier* spans -- the trace is decided fragmented. Mitigation: `decision_wait > dns.interval + propagation + slowest_p99.9`; minimize scale events with conservative HPA; consider PDB to limit voluntary disruptions.

18. **Auto-instrumentation injection sets `OTEL_PROPAGATORS` -- and you can't easily override it per pod.** The OTel Operator's `Instrumentation` CRD's `propagators:` field is set globally per CR, then injected into every annotated pod's env. If service X needs `b3multi` for legacy interop and service Y needs only `tracecontext`, you either run two `Instrumentation` CRs (one per namespace or per labeled set) or set `OTEL_PROPAGATORS` explicitly on the Deployment and accept that the CR's value is the default-overridable one (env vars set on the container override the init-container-injected ones in K8s precedence).

---

## Decision Frameworks

### 1. Head sampling vs Tail sampling vs Both

| Goal | Sampling |
|------|----------|
| Predictable cost, simple architecture, no Collector tail tier | **Head only** at 10-25% |
| Catch all errors and slow traces, accept Collector complexity | **Tail at gateway**, head at 100% in SDK |
| Best of both, scale-friendly | **Head 10-25% in SDK + Tail at gateway with errors/slow/baseline policy** (the production combo) |
| Highest scale (millions of req/sec) | Push tail sampling to backend (Honeycomb Refinery, Tempo TraceQL search, Datadog Live Search) |
| Compliance / regulated (audit every transaction) | **No sampling at all** -- accept the cost; cap retention aggressively instead |

### 2. Tempo vs Jaeger vs X-Ray vs Vendor APM

| Situation | Pick |
|-----------|------|
| Grafana stack, K8s, want cheap retention, structural queries | **Tempo** |
| Existing Cassandra/ES operations, need sub-second cold queries | **Jaeger** |
| AWS-native, want managed simplicity, fine with weaker query language | **X-Ray (via ADOT)** |
| Want best-in-class polish, code profiling, RUM stitching, deployment markers, willing to pay | **Datadog APM or Dynatrace** |
| High-cardinality slicing as the primary use case, SLO-driven analytics-database org | **Honeycomb** |
| Multi-vendor / no-lock-in / migration in flight | **Tempo as primary + vendor as secondary** via Collector fan-out |

### 3. Span events vs Span links vs Separate spans

| Situation | Use |
|-----------|-----|
| Sub-step is fast (< 100us), no duration needed, single-instant marker | **Span event** |
| Sub-step has duration that should appear on the waterfall, has its own attribute set | **Child span** |
| Cross-trace reference (batch in, fan-out, async retry, follow-up trace) | **Span link** (NOT child) |
| Async producer/consumer across queue with hours of dwell time | **Two traces, linked** -- producer span ends; consumer span is root in new trace, links to producer |
| Exception with stack trace inside an in-progress operation | **`record_exception` -> auto-emits `exception` event** |
| Retry attempts within one logical operation | **Events** named `retry.attempt` (cheap) OR child spans (rich) -- depends on whether you want each attempt on the waterfall |

### 4. Auto-instrumentation vs Manual instrumentation

| Situation | Use |
|-----------|-----|
| Net-new service or fleet-wide rollout | **Auto first** -- HTTP, DB, gRPC, queue coverage on day one |
| Business operations (checkout, payment, fraud check) | **Manual** -- libraries can't see your business logic |
| Custom transport (proprietary RPC, internal bus) | **Manual** -- write spans + propagation yourself |
| Async boundary (Kafka, SQS) with supported library | **Auto if available**; the library injects/extracts trace context in headers |
| Async boundary with custom queue (Redis pubsub, internal Kinesis client) | **Manual propagation** -- inject/extract trace context yourself |
| Go services | **Manual + middleware** (otelhttp, otelgrpc, otelpgx); eBPF auto-instrumentation for higher coverage |
| Production answer for 90% of services | **Auto for breadth + manual for the spans that matter** |

### 5. Spanmetrics connector vs explicit RED instrumentation

| Situation | Use |
|-----------|-----|
| Standard HTTP/gRPC/DB span coverage from auto-instrumentation, want RED metrics for free | **Spanmetrics connector** -- lets the trace stream pay for the metric stream |
| Need custom dimensions on metrics that aren't span attributes | **Explicit instrumentation** -- the metric labels you want aren't on the span |
| Need to record metrics for spans that are tail-sampled out | **Explicit instrumentation** -- spanmetrics after tail sampling sees only kept traces |
| High-cardinality metric dimensions (user-id, request-id) | **Neither -- design out the cardinality**; if you really need it, use traces with TraceQL Metrics or Honeycomb |
| Cost-conscious, willing to lose 5% of metric fidelity | **Spanmetrics with conservative dimensions** (3-5 bounded attributes only) |

### 6. Where to run the spanmetrics connector

| Topology | Tradeoff |
|----------|----------|
| In the **agent** (DaemonSet), pre-tail-sampling | Sees 100% of traces -> metric accuracy is full; exemplars often point at sampled-out traces -> click-through 404s |
| In the **front gateway**, pre-tail-sampling | Same as agent but centralized; doesn't help with exemplar 404s |
| In the **tail-sampling pool**, post-tail-sampling | Exemplars always resolve; metric counts are ~5% of true traffic (only sampled traces) -- fine for relative trends, useless for absolute volumes |
| **Two parallel pipelines**: spanmetrics in agent (for accurate metrics), tail-sampled traces in gateway (for storage) | Best of both; double the OTel pipeline complexity |
| Tempo's metrics-generator subsystem | Same problem as post-tail-sampling spanmetrics; runs at Tempo, not Collector |

The most-common-90%-answer: **spanmetrics in the tail-sampling pool, post-sampling**, with documented expectation that absolute span counts are sampling-rate-divided. For absolute traffic counts, fall back to standard request-count metrics.

---

## Why Distributed Tracing Matters Strategically

Metrics tell you something is wrong; logs tell you what an individual machine said about it; traces tell you the *story of one request* across the entire system. None of the other two signals can do this. The reason RED, USE, and the four golden signals all exist is that they're aggregations over an underlying event population -- and the underlying event population, the one you actually need when an incident is mysterious, is the trace.

Until 2018-ish, distributed tracing was niche -- expensive vendors, fragile open-source backends, no standard wire format. By 2026 the picture has flipped: W3C Trace Context is the universal flight-strip format; OTel is the universal SDK; Tempo proves trace storage can be cheap; spanmetrics proves traces can be the *source* of metrics rather than the other way around. The cost curve has bent enough that "instrument every service, sample errors at 100%, store 30 days on object storage" is a tractable production budget for mid-size orgs.

The single most important ROI-shaping decision you'll make in tracing is **the sampling stack** (head + tail + policy mix). Get it right and traces are the cheapest, highest-leverage observability signal in your fleet. Get it wrong and they're the most expensive thing on your bill, with most of the spend going toward duplicate successful checkout traces nobody will ever query.

---

## The 10 Commandments of Distributed Tracing

1. **Thou shalt set span.kind correctly.** SERVER on inbound, CLIENT on outbound, PRODUCER/CONSUMER on async, INTERNAL only for in-process work. The service map depends on it.
2. **Thou shalt propagate W3C Trace Context everywhere, including across queues and through ingresses.** The flight strip must reach every controller; otherwise the trace fragments and the recordings can't be reassembled.
3. **Thou shalt run head sampling generously (10-25%) and tail sampling at the gateway.** Pure head loses errors; pure tail is too expensive at scale; the combo is the production answer.
4. **Thou shalt set `decision_wait` longer than thy slowest service's p99.9.** Otherwise slow traces are decided on incomplete data and silently fragmented.
5. **Thou shalt put the `loadbalancing` exporter in front of the tail-sampling pool, with a headless StatefulSet for stable pod identity.** Without sticky routing, tail sampling makes incoherent decisions on partial traces.
6. **Thou shalt NOT use SimpleSpanProcessor in production.** BatchSpanProcessor everywhere. A synchronous exporter blocking the application thread is a production-disaster waiting to happen.
7. **Thou shalt audit thy spanmetrics `dimensions` before deploying.** Every attribute becomes a metric label; `user.id` is a cardinality bomb; only bounded attributes (`http.method`, `http.status_code`, `http.route` templated) belong there.
8. **Thou shalt use `record_exception` to attach stack traces and `set_status(ERROR)` to flip the span red.** A slow successful span is not an error; an exception-thrown span is. Don't conflate them.
9. **Thou shalt enable exemplars on every histogram derived from spans, and configure Grafana data sources for click-through.** Three-signal correlation is the entire point of the unified telemetry model; missing the data-source links wastes the architecture.
10. **Thou shalt write structural TraceQL queries (`>>`, `>`) and not just tag searches.** "Show me traces where auth called user which then errored" is the killer feature; relying only on tag-based search is leaving Tempo's value on the table.

---

## Further Reading

- [OpenTelemetry Concepts -- Traces](https://opentelemetry.io/docs/concepts/signals/traces/) -- the canonical span anatomy reference
- [W3C Trace Context Specification](https://www.w3.org/TR/trace-context/) -- the wire-format spec; sections 3 and 4 are the must-reads
- [W3C Baggage Specification](https://www.w3.org/TR/baggage/) -- the userland-key-value sibling propagator
- [OTel Tail Sampling Processor README](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/tailsamplingprocessor/README.md) -- every policy type with config examples
- [OTel Spanmetrics Connector README](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/connector/spanmetricsconnector/README.md) -- the trace-to-metrics bridge
- [Grafana Tempo Documentation](https://grafana.com/docs/tempo/latest/) -- TraceQL, storage architecture, metrics-generator
- [TraceQL Language Reference](https://grafana.com/docs/tempo/latest/traceql/) -- span selectors, structural operators, aggregates
- [Honeycomb -- Sampling Documentation](https://docs.honeycomb.io/manage-data-volume/sample/) -- the most practitioner-oriented sampling explainer
- [OTel Blog -- Tail Sampling: Why, How, and What to Consider](https://opentelemetry.io/blog/2022/tail-sampling/) -- the architecture writeup
- Repo cross-references: [OpenTelemetry Fundamentals](./2026-04-27-opentelemetry-fundamentals.md), [Kubernetes Metrics Stack](../prometheus/2026-04-26-kubernetes-metrics-stack.md), [Grafana Fundamentals](../grafana/2026-04-24-grafana-fundamentals.md), [Alertmanager & Alerting Strategy](../prometheus/2026-04-23-alertmanager-alerting-strategy.md), [PromQL Deep Dive](../prometheus/2026-04-21-promql-deep-dive.md), [Prometheus Fundamentals](../prometheus/2026-04-20-prometheus-fundamentals.md)
