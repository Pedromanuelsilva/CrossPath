# CrossPath for ManifoldCF — Implementation Plan

## Goal

Build **CrossPath**, a containerized **Tika-compatible proxy**, to sit between **Apache ManifoldCF `tikaservice`** and external extractor services.

The proxy must expose the HTTP contract expected by ManifoldCF’s remote Tika service connector:

- `PUT /meta`
- `PUT /tika`
- `PUT /detect/stream`

ManifoldCF’s `tikaservice` connector calls those endpoints and expects Tika-compatible behavior, with `/meta` available as JSON when requested and `/tika` available as plain text by default.

The proxy will internally route documents to:

- **Docling Serve** for preferred formats
- **Apache Tika Server** for fallback and MIME detection

This service is **not** responsible for crawling, ACL enforcement, or indexing into OpenSearch. Those remain in ManifoldCF and downstream OpenSearch security/DLS. OpenSearch supports LDAP/AD-backed authorization and document-level security. 

---

## Scope

### In scope
- Tika-compatible HTTP API
- Routing to Docling or Tika
- Routing to custom extractors for recognized document patterns
- Fallback logic
- Request buffering/spooling
- Metadata normalization
- Plain-text extraction normalization
- Health endpoints
- Docker container
- Tests
- README

### Out of scope
- Direct OpenSearch indexing
- Embeddings/vectorization
- OCR-heavy workflows
- Async queueing
- Custom ManifoldCF connector
- Replacing ManifoldCF crawl/state/ACL logic

---

## Acceptance Criteria

### API compatibility
- [ ] `PUT /meta` accepts raw file bytes
- [ ] `PUT /meta` preserves Tika-compatible content negotiation and can return JSON when requested
- [ ] `PUT /tika` accepts raw file bytes
- [ ] `PUT /tika` returns Tika-compatible output, defaulting to plain text
- [ ] `PUT /detect/stream` accepts raw file bytes
- [ ] `PUT /detect/stream` returns detected MIME type as plain text
- [ ] `GET /healthz` returns process-alive status
- [ ] `GET /readyz` returns readiness based on backend availability checks

### HTTP behavior
- [ ] `200` used for successful extraction with body
- [ ] `204` used when request is valid but no extractable output exists
- [ ] `422` used for unsupported/encrypted/rejected documents
- [ ] `500` used for backend/proxy processing failures in line with Tika-compatible endpoint behavior

### Routing
- [ ] `/detect/stream` uses Tika detection by default
- [ ] `.pdf`, `.docx`, `.pptx`, `.xlsx`, `.csv`, `.md`, `.html`, `.xhtml` try Docling first
- [ ] legacy formats like `.doc`, `.ppt`, `.xls`, `.rtf`, `.msg` use Tika first
- [ ] known document patterns can override extension-only routing and select custom extractors
- [ ] document-specific rules are defined in code via matchers/handlers rather than an external rule store
- [ ] a lightweight detection function can classify the file before final route selection
- [ ] if Docling does not support a file or fails for a preferred format, fallback to Tika
- [ ] Tika-primary formats do not fall back to Docling
- [ ] custom extractor routes define explicit fallback behavior
- [ ] fallback behavior is logged and measurable

### Buffering
- [ ] request body is read once
- [ ] small files buffered in memory
- [ ] large files spooled to temp file
- [ ] same buffered input can be reused for multiple backend calls
- [ ] temp files are cleaned up

### Normalization
- [ ] all backend outputs are normalized into one internal model
- [ ] `/tika` preserves Tika-compatible response semantics and plain-text output by default
- [ ] `/meta` preserves Apache Tika-compatible response semantics, including negotiated JSON output
- [ ] Docling metadata is transformed to match Tika output format
- [ ] proxy traceability is carried in headers and logs without breaking Tika response compatibility
- [ ] line endings normalized to `\n`
- [ ] UTF-8 output guaranteed for plain-text `/tika` responses

### Observability
- [ ] structured JSON logs
- [ ] request ID per request
- [ ] metrics for request counts, backend selection, fallback count, latencies, failures

### Deployment
- [ ] Dockerfile builds runnable container
- [ ] `docker-compose.yml` includes proxy + docling + tika example
- [ ] configuration handled via environment variables

### Testing
- [ ] unit tests for routing, normalization, status mapping, buffering
- [ ] integration tests for `/meta`, `/tika`, `/detect/stream`
- [ ] integration test for Docling failure → Tika fallback
- [ ] integration test for Tika failure handling with Tika-compatible status/response behavior

---

## Architecture

### External flow
`ManifoldCF tikaservice -> Proxy -> Docling Serve and/or Tika Server`

> Note: Docling is only used as the primary backend for preferred formats. If Docling does not support a file or fails, the proxy falls back to Tika. Tika-primary formats are not routed back to Docling.

### Internal flow
1. Receive request
2. Buffer body once
3. Detect route
4. Call backend
5. If needed, fallback to second backend
6. Normalize result
7. Format response in Tika-compatible shape
8. Return to ManifoldCF

## Locked Implementation Decisions

- Runtime: Python 3.13+
- Web framework: FastAPI
- Application server: Uvicorn or Gunicorn with Uvicorn workers
- Deployment target: Docker container
- Backend connectivity: internal network access to Docling Serve and Tika Server
- Concurrency model: use multiple worker processes, but keep worker counts conservative because document parsing is expensive and the external backends are likely to be the throughput bottleneck
- Routing input rule: use filename/extension when reliably available from the inbound request context; otherwise default to Tika-first handling
- Initial Docling integration target: `POST` to the official Docling Serve conversion endpoint, starting from `/v1/convert/source` and adapting only if deployment-specific versioning or routing requires a different base path
- Core planned capability: support document-specific routing and custom extractors for known document structures while preserving Tika-compatible outward behavior
- Rule definition model: implement document-specific routing in code through a classifier/matcher function and a registry or dictionary of routing decisions
