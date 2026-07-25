# Distributed Tracing in NIXL — Architecture Design

**Status:** draft for review · **Audience:** NIXL core, plugin owners, Dynamo/vLLM integrators

This document specifies request-scoped distributed tracing for NIXL: one trace context per
request, created once (or supplied by the application), propagated across threads and across
the sender/receiver boundary, with per-phase timestamps keyed to that context and exported to
a vendor-neutral backend (OpenTelemetry).

It is the architecture spec asked for on the tracing thread. It is deliberately separate from
the already-merged `nixl::trace` facade, which is an in-process span API and does not attempt
correlation across processes.

## 1. What exists today

Merged in `main` (PRs #1765, #1845, #1852, #1867, #1882):

| Piece | Where | What it does |
| ----- | ----- | ------------ |
| `nixl::trace` facade | `src/core/tracing/` | RAII spans + markers, composite fan-out to N backends, owned by `nixlAgent` (no singleton) |
| Backend plugin contract | `src/core/tracing/trace_plugin.h` | `libtrace_backend_<name>.so`, `dlopen`'d on demand by `nixlPluginManager` |
| NVTX backend | `src/plugins/tracing/nvtx/` | Nsight Systems timeline; auto-enabled under `nsys` |
| Cross-**thread** correlation | `Tracer::pushCorrelationId()`, `CorrelationScope` | Links `postXferReq` (caller thread) to the `xfer.complete` marker (polling thread) |
| Instrumented call sites | `src/core/nixl_agent.cpp` | 14 call sites across the agent API, spans + typed attributes |

What it deliberately does **not** do:

- The correlation id is `reinterpret_cast<uint64_t>(req_hndl)` — a **process-local** handle. It
  is not unique across processes and is reused after `releaseXferReq`.
- Nothing crosses the wire. Sender and receiver timelines are two unrelated recordings.
- Spans are **scope-shaped**: they measure the synchronous call. The in-flight interval
  (submit → remote completion) is not a span; completion is only a marker, stamped when the
  application happens to poll `getXferStatus`.
- Attribution is per **function**, not per **request**.

Those three gaps are exactly what this design addresses. NVTX keeps its current role:
measuring NIXL's own overheads on a profiler timeline.

## 2. Requirements

| # | Requirement | Source |
| - | ----------- | ------ |
| R1 | Tracing is request-scoped and correlation-first, not function-scoped | Ovidiu |
| R2 | One id per request, generated once or passed in by the application | Ovidiu |
| R3 | Propagated across threads **and** across sender/receiver on the wire | Ovidiu |
| R4 | Per-phase timestamps emitted by the **backend** and keyed to that id | Ovidiu |
| R5 | Accurate in-flight duration, independent of application poll cadence | Ovidiu (vLLM connector experience) |
| R6 | Functional equivalence with Amazon PR #1460, including interop with the libfabric immediate-data transfer id | Ovidiu, Adit |
| R7 | Extensible to all plugins; UCX first; explicit story for every backend | Ovidiu |
| R8 | A generic, industry-standard export path — OpenTelemetry — not only NVTX/Nsight | Adit, Amazon |
| R9 | End-to-end: an id injected by Dynamo, carried through vLLM, consumed by NIXL, so an OpenAI request can be followed to the NIXL transfers it caused | Adit, Dynamo team |
| R10 | Permissively licensed dependencies only (Apache-2.0/MIT/BSD); no GPL | Adit (#1460 licensing) |
| R11 | Zero cost when disabled; no public ABI break; peers on mixed NIXL versions interoperate | NIXL policy |

**Non-goals.** Continuous always-on tracing of every transfer at full detail (sampling is
part of the design); device/in-kernel tracing; replacing telemetry (numeric metrics stay in
`nixl::telemetry`); replacing NVTX; defining Dynamo's or vLLM's own span trees.

## 3. Design overview

```
Dynamo ingress ──traceparent──► vLLM connector ──trace ctx──► nixlAgent.postXferReq
                                                                  │
                                        ┌─────────────────────────┴──────────────────────┐
                                        │ core: trace ctx stored on nixlXferReqH          │
                                        │       phase timestamps keyed to ctx             │
                                        └───────┬─────────────────────────────┬───────────┘
                                                │ opt_b_args (ctx)            │ phase sink
                                        ┌───────▼──────────┐          ┌───────▼──────────┐
                                        │ plugin (UCX/…)   │          │ trace backends   │
                                        │ ctx on the wire  │          │ otel / nvtx /    │
                                        │ CQ-time stamps   │          │ chakra           │
                                        └───────┬──────────┘          └──────────────────┘
                                                │ wire: 26-byte context record
                                        ┌───────▼──────────┐
                                        │ receiver agent   │ remote-parent spans, same trace id
                                        └──────────────────┘
```

Four independent pieces, in dependency order:

1. **Trace context** — a W3C-compatible value type, plus a public way for the application to
   supply one and for NIXL to generate one.
2. **Request-scoped core plumbing** — context stored per request, phase timestamps emitted
   against it, correlation scopes keyed by it instead of by pointer.
3. **Wire propagation** — a compact, versioned, capability-negotiated context record, carried
   per plugin.
4. **Export** — an OpenTelemetry backend plugin behind the existing `nixl::trace` contract.

Pieces 1, 2 and 4 already deliver R9 on the sender side with **no wire change at all**: if
vLLM hands NIXL the Dynamo trace context, NIXL's spans become children of the Dynamo request
span, and an OpenAI request can be followed into NIXL post/complete timings. Piece 3 is what
adds the receiver side. This is the recommended delivery order.

## 4. Trace context

```c++
namespace nixl::trace {
struct TraceContext {                 // 25 bytes of W3C fields
    std::array<std::uint8_t, 16> traceId{};   // W3C trace-id, all-zero = invalid
    std::array<std::uint8_t, 8>  spanId{};    // W3C span-id of the *parent* span
    std::uint8_t flags{};                     // bit0 = sampled (W3C trace-flags)
    [[nodiscard]] bool valid() const noexcept;
    [[nodiscard]] bool sampled() const noexcept;
};
}
```

**Why W3C Trace Context.** It is what OpenTelemetry propagates by default, what Dynamo and
vLLM will already have in-process for an incoming OpenAI request, and it is the only format
every observability vendor accepts. A NIXL-private id would have to be joined to the rest of
the inference trace by hand — precisely the hand-rolled instrumentation this effort is meant
to remove.

Note that OpenTelemetry has **no binary propagator** (the spec dropped it; it is HTTP-header
oriented). NIXL therefore defines its own compact binary encoding of the same fields — 16 + 8
+ 1 bytes, big-endian, no text — and converts to/from `traceparent` at the API boundary.
Applications that only have a string pass the 55-character `traceparent`; NIXL parses it.

### 4.1 Where the context comes from

Three sources, in priority order:

1. **Application-supplied** (R2, R9). A new optional field on `nixl_opt_args_t`, accepted by
   `createXferReq` / `makeXferReq` / `postXferReq` and by `genNotif`.
2. **NIXL-generated** when tracing is active and the application supplied nothing: a fresh
   random trace id per request, so NIXL traces stand alone and are still request-scoped.
3. **Absent** — tracing off, or sampling said no. No context, no wire bytes, no cost.

For the Dynamo path the application-supplied route is the one that matters. It is worth being
explicit about what vLLM does **today**: the NIXL connector already passes its request id as
the notification payload (`make_prepped_xfer(..., notif_msg=notif_id)`), and NIXL treats that
blob as opaque. That is evidence the id exists and already reaches the peer — but the notif
blob is the application's own channel, it is only delivered at completion, and NIXL cannot
key phase timestamps to it. A first-class field is what turns it into a trace.

### 4.2 Public API shape

```c++
struct nixlAgentOptionalArgs {
    // ... existing fields ...
    /** Trace context to attach to this request; empty = NIXL decides.
     *  Accepts a 25-byte binary context or a W3C `traceparent` string. */
    nixl_blob_t traceContext;
};
```

Appending a defaulted field to a C++ struct matches how `nixl_opt_args_t` has been extended
before (`notif`, `metadataLabel`, `customParam`) and keeps source compatibility. `nixlAgentConfig`
is **not** touched, per the ABI rule the tracing work already follows. Bindings:

- Python: `notif_msg`-style keyword (`trace_context=`) on `make_prepped_xfer`, `initialize_xfer`,
  `transfer`, `send_notif`; this is the surface the vLLM connector will use.
- Rust: `OptArgs::set_trace_context()`.
- Both also get a getter for the context NIXL generated, so an application can log/join it.

## 5. Request-scoped core plumbing

- `nixlXferReqH` gains a `TraceContext` member next to `notifMsg` and the existing
  `nixl_xfer_telem_t`. It is set at request creation and is immutable for the request's life.
- The existing `NIXL_TRACE_CORRELATION_SCOPE` is re-keyed from the handle pointer to the
  request's context, which makes cross-thread correlation (already merged) and cross-process
  correlation the same mechanism instead of two.
- `nixl_opt_b_args_t` gains the context so `prepXfer`/`postXfer` hand it to the plugin; the
  plugin stores it on its own `nixlBackendReqH`, which is what lets its progress thread
  attribute a CQ event to a request.

### 5.1 Phases, and why they must be stamped in the backend

This is the heart of R4/R5. Today a completion is stamped when the application calls
`getXferStatus` and happens to see success, so the measured in-flight time includes the
application's poll latency — the reason accurate per-phase breakdowns were not obtainable in
the vLLM connector without hand-rolled code. The fix is that whoever *first observes* an event
stamps it, on the thread that observes it:

| Phase | Stamped by | Notes |
| ----- | ---------- | ----- |
| `request.created` | core, `createXferReq`/`makeXferReq` | descriptor prep cost |
| `post.enter` / `post.submitted` | core, `postXferReq` | already measured as `postDuration` |
| `wire.submitted` | plugin, at the transport call | per rail/QP for striped transfers |
| `wire.completed` | plugin, when the CQ/AM event is reaped | **the accurate end of in-flight** |
| `status.observed` | core, `getXferStatus` | poll latency = `status.observed − wire.completed` |
| `notif.sent` / `notif.received` | plugin | receiver-side arrival |
| `remote.write.observed` | plugin (receiver) | where the transport reports it (libfabric) |

Two API additions follow from the table:

1. **Retroactive, explicitly-timestamped spans.** An in-flight interval cannot be an RAII
   scope: it begins on one thread and ends on another, and both endpoints are known only
   after the fact. `TraceBackend` gains `emitSpan(name, kind, ctx, start, end, attrs)`
   alongside the existing `beginSpan`. Backends that cannot place a span in the past (NVTX)
   ignore it or degrade to a marker; offline/OTLP backends record it exactly.
2. **A plugin-facing phase sink.** Trace backends are separate `.so`s from data plugins, so
   plugins cannot call the facade directly. `nixlBackendInitParams` gains an optional pointer
   to an abstract sink (`recordPhase(ctx, phase, timestamp, attrs)`), appended at the end of
   the struct and null when tracing is off; the sink is a pure interface owned by core, so a
   plugin built against an older header keeps working. `NIXL_PLUGIN_API_VERSION` is bumped.

Timestamps use the existing `nixlTime::nixlDuration` TSC/`cntvct` stopwatch already used by
telemetry, converted once to wall clock at export, so hot-path cost stays at a counter read.

### 5.2 Sampling

Head-based, decided once per request, recorded in `flags` bit 0 and honoured by every peer:

- Application-supplied context: NIXL respects the incoming sampled bit (Dynamo decides).
- NIXL-generated: `NIXL_TRACE_SAMPLE_RATIO` (default `0`, i.e. off unless asked).

Unsampled requests write no wire bytes and take one branch. This is what makes it safe to
leave the feature compiled in by default.

## 6. On-the-wire propagation

### 6.1 Record format

```
byte 0      : version (0x01)
byte 1      : flags   (bit0 sampled)
bytes 2-17  : trace-id (16B, big-endian)
bytes 18-25 : span-id  (8B, big-endian) — the sender's span, becomes the receiver's parent
```

26 bytes, fixed. Version-first so an unknown version is skipped, not misparsed.

### 6.2 Compatibility

A peer running an older NIXL must not see corrupted control messages (R11). Two mechanisms,
chosen per plugin by what its format allows:

- **Append-only SerDes.** `nixlSerDes` reads strictly positionally by tag and simply stops
  after the tags a reader knows. Appending `addStr("tctx", ...)` **after** the existing
  `name`/`msg` tags in UCX notifications is therefore transparent to an old receiver. This is
  the cheapest safe carrier and the reason UCX is first.
- **Capability negotiation** where the format is a packed struct with no version field
  (libfabric `BinaryNotification`). The peer advertises trace-context support during its
  existing handshake (libfabric handshake SerDes already carries `idx`/`name`/`has_conn`/`conn`
  and can take one more appended tag), and NIXL only sends the record to peers that
  advertised it.

Both are per-plugin decisions, which is why the backend engine grows a capability query
(`supportsTraceContext()`, defaulting to false) rather than the core assuming propagation.
Core degrades cleanly: sender-side spans and phases still work, receiver spans are simply
absent, and that is reported once per peer at debug level.

### 6.3 Per-plugin carriers

| Plugin | Carrier | Receiver-side event | Effort |
| ------ | ------- | ------------------- | ------ |
| **UCX** | `tctx` tag appended to the notification SerDes; the AM header (currently `hdr=nullptr, hdr_len=0`) is available as a second option | AM callback on notification arrival. Plain `ucp_put_nbx`/`ucp_get_nbx` produce **no** receiver event and carry no immediate data | Low; first target (R7) |
| **libfabric** | record in `BinaryNotification` fragment 0 behind handshake capability; the existing 32-bit `imm_data` (`4b type / 8b agent / 16b xfer_id / 4b seq`) is **full** and stays as-is | `FI_REMOTE_CQ_DATA` on remote write — the one transport that reports data arrival | Medium; keeps #1460's mechanism intact (R6) |
| **gpunetio (DOCA)** | extend the `DOCAS…DOCAE` notification envelope | notification RQ poll; RDMA WQEs carry no immediate data today | Medium |
| **mooncake, uccl** | `notify_msg_t.msg` (already a SerDes blob for uccl) | library-internal; data path needs upstream support | Low for notif, blocked for data path |
| **posix, obj, azure_blob, hf3fs, gds_mt, cuda_gds, gusli, infinia** | none — no NIXL peer | n/a | Context stays local; spans are still emitted |

**The important asymmetry.** For plain RDMA write/read there is no receiver-side event at all
except in libfabric. So "receiver spans" mean, precisely: a span at the point the receiver
*learns* about the transfer — notification arrival for UCX/DOCA, remote-write completion for
libfabric. The design does not pretend otherwise, and does not add a synthetic control message
per transfer just to manufacture one; that would trade the overhead NVTX is used to measure
for tracing detail. Where an application already asks for a notification (the vLLM connector
does, on every KV write), the receiver span is free.

### 6.4 Interop with Amazon PR #1460 (R6)

#1460's value was request correlation, not its tracepoint library — and its correlation
mechanism (a transfer id in libfabric immediate data, matched to notifications by expected
completion counts) is **already merged** in `src/utils/libfabric/`. What was libfabric-specific
and GPL-encumbered was the *ingest* side (LTTng-UST). This design keeps the mechanism and
replaces the ingest:

- The 16-bit `xfer_id` stays the transport-local key that joins rail completions to a
  notification. It is *not* widened.
- The 128-bit trace id is carried in the notification and mapped to `xfer_id` on both sides,
  so a rail-level CQ event resolves to a request and therefore to a trace.
- Every event #1460 emitted (`post_write/read/send` begin/end, local and remote completions,
  submitted counts) becomes a phase timestamp on the sink from §5.1, keyed to the trace
  context, exported through Apache-2.0 OpenTelemetry instead of LTTng (R10).

Amazon should be invited to review this section specifically.

## 7. Export backends

### 7.1 OpenTelemetry backend plugin

`libtrace_backend_otel.so`, implementing the existing `nixlTracePlugin` contract — no new
plumbing, and the heavy dependency stays out of `libnixl` and is `dlopen`'d only when
`NIXL_TRACE_BACKENDS=otel`.

- **Library:** `opentelemetry-cpp` (Apache-2.0; v1.28.0, July 2026). Satisfies R10.
- **Transport:** OTLP/HTTP is the default (protobuf ≥ 3.21.6, libcurl, nlohmann/json, zlib).
  OTLP/gRPC additionally pulls gRPC + abseil and is a build option, not the default; NIXL
  already vendors abseil, so the gRPC path is feasible but heavier.
- **Build gating:** same shape as the Prometheus telemetry exporter — system package first,
  `cmake.subproject` wrap as fallback, plugin silently skipped when unavailable, hard error
  only when explicitly requested. Added to `trace_backends`.
- **Config:** honour the standard `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME`,
  `OTEL_TRACES_SAMPLER*` variables rather than inventing NIXL names, so a Dynamo deployment
  configures NIXL the same way it configures everything else. `NIXL_TRACE_BACKENDS=otel` stays
  the on/off switch.
- **Hot path:** the batch span processor exports on its own thread; span creation is an
  allocation and a memcpy of attributes. Nothing blocks a transfer.
- **Span model:** sender span kind `PRODUCER`, receiver `CONSUMER`, receiver's parent taken
  from the wire context (remote parent), plus a span link where a receiver serves several
  requests. Attributes namespaced `nixl.*` (`nixl.agent`, `nixl.remote_agent`, `nixl.backend`,
  `nixl.op`, `nixl.bytes`, `nixl.desc_count`, `nixl.mem_type`, `nixl.status`). One span per
  **request**, never per descriptor; rail/QP fan-out becomes events on that span.
- **Packaging:** like the DOCA exporter, ship as an optional system-side plugin rather than
  bundling protobuf/curl into the `nixl-cu12` wheel; `auditwheel` excludes accordingly.

**Alternative considered:** a dependency-free OTLP/HTTP+JSON emitter (a few hundred lines,
no protobuf). Cheaper to package, but it re-implements batching, retry, resource detection and
schema versioning, and diverges from the ecosystem the moment semconv moves. Rejected as the
primary path; it remains a fallback if the dependency footprint proves unacceptable in the
wheel, and the plugin boundary means that decision can be revisited without touching core.

### 7.2 NVTX and Chakra

NVTX is unchanged and stays the tool for NIXL's own overheads: it gains the real trace id as
its correlation payload (instead of a pointer), which makes an Nsight timeline joinable to an
OTLP trace by id. Chakra remains the offline execution-trace backend; a wire-propagated,
globally unique id is exactly what its `chakra_trace_link` step needs, so this design unblocks
it rather than competing with it.

## 8. Delivery plan

Each item is a reviewable PR under the repo's 500-line limit, stacked in order. Phases 1 and 2
are independently useful and are what unblock the Dynamo ask.

**Phase 1 — sender-side end-to-end (no wire change).**

1. `TraceContext` type, `traceparent` parse/format, unit tests.
2. `nixl_opt_args_t::traceContext`, generation + sampling policy, stored on `nixlXferReqH`,
   correlation scopes re-keyed off it.
3. Python and Rust bindings for supplying and reading the context.
4. OpenTelemetry backend plugin: build gating, span emission, OTLP/HTTP export, gtest against
   a stub collector.
5. `docs/tracing.md` update; e2e test asserting a NIXL span joins an application-supplied trace.

**Phase 2 — accurate phases.**

6. `TraceBackend::emitSpan()` (explicit timestamps) + facade tests.
7. Plugin-facing phase sink in `nixlBackendInitParams`, `NIXL_PLUGIN_API_VERSION` bump.
8. Context through `nixl_opt_b_args_t` onto backend request handles.
9. UCX phase stamping at submit and CQ reap; in-flight span asserted independent of poll cadence.

**Phase 3 — wire propagation.**

10. Wire record encode/decode + `supportsTraceContext()` capability + core degradation path.
11. UCX carrier (appended SerDes tag) + two-process gtest asserting one trace id, two agents.
12. libfabric carrier + `xfer_id` mapping (#1460 equivalence), reviewed with Amazon.
13. gpunetio carrier; mooncake/uccl notification-only carrier.

**Phase 4 — breadth.** Chakra backend consuming the same ids; overhead benchmark in CI
(sampled and unsampled); nixlbench integration.

## 9. Open questions for review

1. **Who owns id generation** when the application supplies nothing — NIXL, or should NIXL
   refuse to invent one and stay purely a propagator?
2. **Default sampling ratio.** Proposal: off. Is an always-on 1-in-N default wanted for
   production Dynamo deployments?
3. **Public vs internal API.** `nixl::trace` is internal today. The context type appears in
   `nixl_opt_args_t`, which is public. Do we expose the struct, or keep the public surface a
   `traceparent` string/blob only (current proposal)?
4. **Receiver spans without notifications.** Accept the gap for plain RDMA, or offer an opt-in
   "trace control message" for users who want receiver visibility and will pay for it?
5. **OTLP transport default** — HTTP (lighter deps) vs gRPC (what most collectors expect).
6. **Wheel policy** for the OTel plugin: optional system package (proposal) or bundled?
7. **Semantic conventions.** Should `nixl.*` attributes be proposed upstream to OpenTelemetry
   semconv, or aligned with whatever Dynamo already emits?
8. **Ownership.** Core (context, phases, API) vs plugin owners (per-plugin carriers) —
   Phase 3 items 11–13 need named owners per plugin.

## 10. Appendix: which review feedback is already in the merged PRs

| Feedback | State |
| -------- | ----- |
| Don't couple NIXL to one tool; make the backend pluggable | **Done** — plugin contract, NVTX is just the first |
| Correlate spans emitted on different threads | **Done** — `pushCorrelationId`/`CorrelationScope`, used for post → completion |
| Backend-agnostic correlation API (not an NVTX concept) | **Done** — id lives in the facade, NVTX renders it as a payload |
| Don't require Nsight to consume traces | **Partly** — the contract allows any backend; this design adds the generic one (R8) |
| Request-scoped id, generated once or supplied by the app | **Not done** — §4, §5 |
| Propagate on the wire, sender ↔ receiver | **Not done** — §6 |
| Per-phase timestamps emitted by the backend, keyed to the id | **Not done** — §5.1 |
| Accurate in-flight duration | **Not done** — §5.1, and the reason the phase sink is plugin-facing |
| Equivalence with #1460 incl. libfabric immediate-data interop | **Not done** — §6.4 |
| Extensibility to other plugins, UCX first | **Not done** — §6.3 |
