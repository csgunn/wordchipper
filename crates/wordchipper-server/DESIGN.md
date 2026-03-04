# wordchipper-server Design

## Overview

A network tokenization service built on wordchipper. Deploy on well-provisioned
machines; clients send text or tokens over the wire and get results back without
needing to integrate the tokenizer library directly.

**Philosophy**: Spend network to avoid software integration complexity on the
client side.

## Inspirational Reference: Have Quick

The [havequick](file:///Users/csg/wyrdward/havequick/) project provides a
strong set of ideas we should draw from as this design matures.

### Five Primitives

Have Quick reduces all composition to five primitives:

```
Socket ──parse──> Protocol ──functor──> TypedValue ──dispatch──> Effect
```

| Primitive | What it does |
|-----------|--------------|
| Socket | Transport (bytes to bytes) |
| Protocol | Parser (bytes to Message[T]) |
| Functor | Transform (Message[A] to Message[B]) |
| Type dispatch | Route by type_hash |
| Composition | Glue pipelines together |

Our protocol adapters are functors in this sense — structure-preserving
transforms between wire representations and canonical types. The service core
is dispatch. The pipeline should be that clean.

### Three-Plate Quality

Truth from cross-validation, not authority:

- **Plate 1: Unit tests** — isolated correctness
- **Plate 2: Fuzz testing** — random input stress
- **Plate 3: Chaos testing** — controlled corruption of middles

We should apply this to the server: unit tests for each layer, fuzz the
adapters with malformed wire data, chaos-test the service under concurrent
load with poisoned inputs and dropped connections.

### 3D UNIX Pipes

UNIX pipes in three dimensions: **typed-data** (what it IS), **context**
(where it came FROM), **intention** (what sender WANTS). When all three agree,
the message flows. Our Content-Type / model / verb triple is a lightweight
version of this. Worth keeping that framing as we add protocols.

### Algebraic Correctness

> A program is a morphism between groups. Correct means the diagram commutes.

If we encode text through the JSON adapter and through the msgpack adapter,
and both produce the same canonical EncodeRequest, and the service produces
the same tokens — the diagram commutes. That's a testable fact. Each adapter
is a plate. Cross-validation between adapters is the scraping.

### Witnesses

> The path is witnessed. Logs, traces, proofs are witnesses.

Request IDs, latency traces, and the metrics counters are witnesses of the
system's behavior. Not afterthoughts — structural participants.

## Architecture

```
 Clients                   Protocol Adapters              Core Service
┌──────────┐              ┌───────────────┐              ┌──────────────┐
│ HTTP/JSON ├──────────────┤  JSON Adapter │──┐           │              │
└──────────┘              └───────────────┘  │           │  Canonical   │
┌──────────┐              ┌───────────────┐  ├──────────►│  Request /   │
│ gRPC     ├──────────────┤ gRPC Adapter  │──┘           │  Response    │
└──────────┘              └───────────────┘  │           │  Types       │
┌──────────┐              ┌───────────────┐  │           │              │
│ msgpack  ├──────────────┤msgpack Adapter│──┘           │      │       │
└──────────┘              └───────────────┘              │      ▼       │
                                                         │  Model       │
                                                         │  Registry    │
                                                         │      │       │
                                                         │      ▼       │
                                                         │  Encoder /   │
                                                         │  Decoder     │
                                                         │  (rayon)     │
                                                         └──────────────┘
```

### Layered Design

The system has three layers:

1. **Protocol Adapters** — thin conversion layers that parse wire format into
   canonical types and serialize responses back. Each adapter implements a
   `FromWire` / `ToWire` trait pair. Adapters are stateless.

2. **Canonical Types** — the internal request/response types that all adapters
   convert to/from. These types are serializable (serde) so they can also go
   directly on the wire as a "native" format. All backend methods accept and
   return canonical types only.

3. **Service Core** — model registry, encoder/decoder dispatch, batch
   parallelism. Stateless per-request; all state is in the pre-loaded model
   registry.

This means adding a new wire format (e.g. flatbuffers, CBOR) requires only
writing a new adapter — no changes to the service core.

## Canonical Types

These are the internal types that cross the adapter/service boundary. They are
serde-serializable so they double as a wire format.

```rust
/// Identifies a loaded model.
type ModelName = String;

// ── Requests ──

struct EncodeRequest {
    model: ModelName,
    text: String,
}

struct BatchEncodeRequest {
    model: ModelName,
    texts: Vec<String>,
}

struct DecodeRequest {
    model: ModelName,
    tokens: Vec<u32>,
}

struct BatchDecodeRequest {
    model: ModelName,
    tokens: Vec<Vec<u32>>,
}

// ── Responses ──

struct EncodeResponse {
    tokens: Vec<u32>,
    count: usize,
}

struct BatchEncodeResponse {
    results: Vec<EncodeResponse>,
}

struct DecodeResponse {
    text: String,
}

struct BatchDecodeResponse {
    results: Vec<DecodeResponse>,
}

struct ModelInfo {
    name: String,
    vocab_size: usize,
    special_tokens: HashMap<String, u32>,
}

struct ModelsResponse {
    models: Vec<ModelInfo>,
}

// ── Errors ──

struct ErrorResponse {
    error: String,
    code: u32,  // mirrors HTTP status semantics
}
```

## Protocol Adapter Trait

```rust
/// Converts from a wire-format request body into a canonical request.
trait FromWire<T> {
    fn from_wire(bytes: &[u8], content_type: &str) -> Result<T, ErrorResponse>;
}

/// Converts a canonical response into wire-format bytes.
trait ToWire<T> {
    fn to_wire(response: &T, accept: &str) -> Result<(Vec<u8>, String), ErrorResponse>;
}
```

The HTTP layer inspects `Content-Type` and `Accept` headers to select the
appropriate adapter. The adapter registry maps MIME types to implementations:

| MIME Type                   | Adapter   | Status    |
| --------------------------- | --------- | --------- |
| `application/json`          | JSON      | v1        |
| `application/msgpack`       | MessagePack | planned |
| `application/protobuf`      | Protobuf  | planned   |
| `application/octet-stream`  | Raw canonical (bincode) | planned |

## HTTP API (v1)

### Discovery

```
GET /v1/models
  → 200  ModelsResponse

GET /v1/models/{model}
  → 200  ModelInfo
  → 404  ErrorResponse (unknown model)
```

### Encode

```
POST /v1/models/{model}/encode
  Body: {"text": "hello world"}
  → 200  EncodeResponse

POST /v1/models/{model}/encode/batch
  Body: {"texts": ["hello world", "foo bar"]}
  → 200  BatchEncodeResponse
```

### Decode

```
POST /v1/models/{model}/decode
  Body: {"tokens": [15339, 1917]}
  → 200  DecodeResponse

POST /v1/models/{model}/decode/batch
  Body: {"tokens": [[15339, 1917], [8134, 3703]]}
  → 200  BatchDecodeResponse
```

### Operations

```
GET /healthz   → 200 {"status": "ok"}
GET /readyz    → 200 {"status": "ready", "models_loaded": 6}
               → 503 {"status": "loading", "models_loaded": 3, "models_total": 6}
GET /metrics   → 200 (prometheus text format)
```

### Content Negotiation

All POST endpoints respect `Content-Type` for request parsing and `Accept` for
response serialization. Default is `application/json` for both. This is how
msgpack and future formats plug in without changing the URL structure.

### Error Responses

All errors return a JSON body (regardless of Accept header for simplicity):

```json
{"error": "model 'foo' not found", "code": 404}
```

HTTP status codes:
- `400` — malformed request, missing fields
- `404` — unknown model
- `422` — valid JSON but invalid tokens (e.g. out of vocab range)
- `500` — internal encoder/decoder error
- `503` — server not ready (models still loading)

## Configuration

A TOML config file specifies which models to load and server settings.

```toml
# wordchipper-server.toml

[server]
bind = "0.0.0.0:8080"
# Worker threads for async HTTP handling.
# Defaults to number of CPUs.
# workers = 8

[server.metrics]
enabled = true
# bind = "0.0.0.0:9090"  # separate metrics port (optional)

# Models to load at startup.
# Each [[model]] entry loads one tokenizer.
# The server will not accept requests until all models are loaded.

[[model]]
name = "cl100k_base"

[[model]]
name = "o200k_base"

[[model]]
name = "o200k_harmony"

[[model]]
name = "r50k_base"

[[model]]
name = "p50k_base"

# Custom / user-trained models can specify a vocab path:
# [[model]]
# name = "my_custom_model"
# vocab_path = "/data/vocabs/my_model.tiktoken"
# pattern = "cl100k_base"  # reuse an existing regex pattern

# Encoder settings applied to all models.
[encoding]
# Use logos DFA accelerators where available.
accelerated_lexers = true
# SpanEncoder variant: "ConcurrentDefault", "PriorityMerge", etc.
span_encoder = "ConcurrentDefault"
```

## Service Core

```rust
struct TokenizerService {
    models: HashMap<String, LoadedModel>,
}

struct LoadedModel {
    info: ModelInfo,
    encoder: Arc<dyn TokenEncoder<u32>>,
    decoder: Arc<dyn TokenDecoder<u32>>,
}

impl TokenizerService {
    fn encode(&self, req: EncodeRequest) -> Result<EncodeResponse, ErrorResponse>;
    fn encode_batch(&self, req: BatchEncodeRequest) -> Result<BatchEncodeResponse, ErrorResponse>;
    fn decode(&self, req: DecodeRequest) -> Result<DecodeResponse, ErrorResponse>;
    fn decode_batch(&self, req: BatchDecodeRequest) -> Result<BatchDecodeResponse, ErrorResponse>;
    fn list_models(&self) -> ModelsResponse;
    fn get_model(&self, name: &str) -> Result<ModelInfo, ErrorResponse>;
}
```

All methods are synchronous and CPU-bound. The HTTP layer calls them via
`tokio::task::spawn_blocking` to avoid starving the async runtime.

Batch methods use rayon internally (via wordchipper's `ParallelRayonEncoder` /
`ParallelRayonDecoder`).

## Planned: gRPC

The canonical types map directly to protobuf messages. A gRPC adapter would:
- Define `.proto` from the canonical types (or generate canonical types from
  `.proto` — either direction works since they're isomorphic).
- Implement `tonic` service trait calling the same `TokenizerService` methods.
- Support server-streaming for large batch responses.

The gRPC server can run on a separate port alongside HTTP, or as a standalone
binary.

## Planned: Streaming

For very large batch requests, a streaming mode:
- Client sends a stream of texts (one per frame / line / message).
- Server returns a stream of token arrays in the same order.
- Works over: HTTP chunked transfer, SSE, gRPC server-streaming, or WebSocket.

The service core method for this:
```rust
fn encode_stream(
    &self,
    model: &str,
    texts: impl Iterator<Item = String>,
) -> impl Iterator<Item = Result<EncodeResponse, ErrorResponse>>;
```

## Crate Structure

```
crates/wordchipper-server/
├── Cargo.toml
├── src/
│   ├── main.rs              # CLI entry point, config loading, startup
│   ├── config.rs             # Config file parsing (TOML)
│   ├── canonical.rs          # Canonical request/response types
│   ├── service.rs            # TokenizerService (core logic)
│   ├── adapters/
│   │   ├── mod.rs            # FromWire / ToWire traits
│   │   └── json.rs           # JSON adapter
│   └── http/
│       ├── mod.rs            # axum router setup
│       ├── routes.rs         # handler functions
│       └── middleware.rs     # content negotiation, metrics, logging
```

## Dependencies

| Crate     | Purpose                              |
| --------- | ------------------------------------ |
| wordchipper | Core tokenizer library             |
| axum      | HTTP framework                       |
| tokio     | Async runtime                        |
| serde     | Serialization for canonical types    |
| serde_json| JSON adapter                         |
| toml      | Config file parsing                  |
| clap      | CLI argument parsing                 |
| tracing   | Structured logging                   |
| metrics + metrics-exporter-prometheus | Observability |

## Implementation Phases

### Phase 1 — MVP
- Config file loading
- Model registry (load at startup)
- Canonical types with serde
- JSON adapter
- HTTP routes (encode, decode, batch, models, healthz)
- Basic request logging

### Phase 2 — Production Hardening
- Prometheus metrics (request count, latency histograms, batch sizes)
- Request size limits
- Graceful shutdown
- Readiness probe (models loaded)
- MessagePack adapter

### Phase 3 — Advanced Protocols
- gRPC service (tonic)
- Streaming encode/decode
- WebSocket adapter

### Phase 4 — Deployment
- Dockerfile
- Helm chart / k8s manifests
- Load testing results
- Client libraries (Python, TypeScript) — thin wrappers over HTTP
