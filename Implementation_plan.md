# Tika Proxy for ManifoldCF — Implementation Plan

## Goal

Build a containerized **Tika-compatible proxy** that sits between **Apache ManifoldCF `tikaservice`** and external extractor services.

The proxy must expose the HTTP contract expected by ManifoldCF’s remote Tika service connector:

- `PUT /meta`
- `PUT /tika`
- `PUT /detect/stream`

ManifoldCF’s `tikaservice` connector calls those endpoints, expects `/meta` as JSON and `/tika` as plain text, and treats `503` as retryable. 

The proxy will internally route documents to:

- **Docling Serve** for preferred formats
- **Apache Tika Server** for fallback and MIME detection

This service is **not** responsible for crawling, ACL enforcement, or indexing into OpenSearch. Those remain in ManifoldCF and downstream OpenSearch security/DLS. OpenSearch supports LDAP/AD-backed authorization and document-level security. 

---

## Scope

### In scope
- Tika-compatible HTTP API
- Routing to Docling or Tika
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
- [ ] `PUT /meta` returns `application/json`
- [ ] `PUT /tika` accepts raw file bytes
- [ ] `PUT /tika` returns `text/plain; charset=utf-8`
- [ ] `PUT /detect/stream` accepts raw file bytes
- [ ] `PUT /detect/stream` returns detected MIME type as plain text
- [ ] `GET /healthz` returns process-alive status
- [ ] `GET /readyz` returns readiness based on backend availability checks

### HTTP behavior
- [ ] `200` used for successful extraction with body
- [ ] `204` used when request is valid but no extractable output exists
- [ ] `422` used for unsupported/encrypted/rejected documents
- [ ] `503` used for backend timeout/unavailable/retryable failures
- [ ] `500` used only for unexpected proxy bugs

### Routing
- [ ] `/detect/stream` uses Tika detection by default
- [ ] `.pdf`, `.docx`, `.pptx`, `.xlsx`, `.csv`, `.md`, `.html`, `.xhtml` try Docling first
- [ ] legacy formats like `.doc`, `.ppt`, `.xls`, `.rtf`, `.msg` use Tika first
- [ ] if Docling does not support a file or fails for a preferred format, fallback to Tika
- [ ] Tika-primary formats do not fall back to Docling
- [ ] fallback behavior is logged and measurable

### Buffering
- [ ] request body is read once
- [ ] small files buffered in memory
- [ ] large files spooled to temp file
- [ ] same buffered input can be reused for multiple backend calls
- [ ] temp files are cleaned up

### Normalization
- [ ] all backend outputs are normalized into one internal model
- [ ] `/tika` always returns canonical plain text
- [ ] `/meta` always returns flattened JSON metadata
- [ ] metadata includes `Content-Type`, `parser_used`, and proxy trace fields
- [ ] line endings normalized to `\n`
- [ ] UTF-8 output guaranteed for `/tika`

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
- [ ] integration test for Tika timeout → `503`

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
