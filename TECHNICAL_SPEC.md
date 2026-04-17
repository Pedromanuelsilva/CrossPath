# Tika Proxy — Technical Specification

## Overview
This document provides detailed technical requirements for implementing the Tika proxy service.

---

## System Integration

### ManifoldCF
- **Version**: Latest (as of implementation)
- **Connector**: "External Tika server" connector (see Apache ManifoldCF official docs)
- **API Contract**: `PUT /meta`, `PUT /tika`, `PUT /detect/stream`
- **Expected Behavior**: Treats `503` as retryable; expects `/meta` response as `application/json`, `/tika` as `text/plain; charset=utf-8`

### Backend Services
- **Docling**: onprem container (docling-serve) — HTTP endpoint, no authentication
- **Tika**: onprem container (apache/tika:latest with OCR) — HTTP endpoint, no authentication

---

## Routing Logic

### Request Type Determination
The proxy determines which backend(s) to call based on file extension and request path.

**For `PUT /detect/stream`:**
- Always call Tika Server for MIME type detection (primary)

**For `PUT /meta` and `PUT /tika`:**

| Extension    | Primary | Fallback |
| ------------ | ------- | -------- |
| `.pdf`       | Docling | Tika     |
| `.docx`      | Docling | Tika     |
| `.pptx`      | Docling | Tika     |
| `.xlsx`      | Docling | Tika     |
| `.csv`       | Docling | Tika     |
| `.md`        | Docling | Tika     |
| `.html`      | Docling | Tika     |
| `.xhtml`     | Docling | Tika     |
| `.doc`       | Tika    | none     |
| `.ppt`       | Tika    | none     |
| `.xls`       | Tika    | none     |
| `.rtf`       | Tika    | none     |
| `.msg`       | Tika    | none     |
| (all others) | Tika    | none     |

### Fallback Behavior
1. **Docling primary formats**: if Docling does not support the file or fails, route to Tika.
2. **Tika primary formats and unknown formats**: do not fall back to Docling.
3. **No backend available**: return appropriate error status (422, 503, or 500).
4. **One backend unavailable** (during /readyz): log and flag; other backend takes requests.
5. **Both backends unavailable**: return 503.

---

## Error Handling & HTTP Status Codes

### Status Code Mapping

| Code | Meaning               | When to Use                                                                                  |
| ---- | --------------------- | -------------------------------------------------------------------------------------------- |
| 200  | OK                    | Successful extraction with content in response body                                          |
| 204  | No Content            | Valid document processed but no extractable content (e.g., blank/image-only PDF without OCR) |
| 400  | Bad Request           | Missing/invalid request headers (e.g., no Content-Type)                                      |
| 422  | Unprocessable Entity  | Document format not supported, encrypted, or rejected by backend                             |
| 500  | Internal Server Error | Proxy bug (not backend unavailability)                                                       |
| 503  | Service Unavailable   | Backend timeout, temporarily unavailable, or both backends down (retryable by ManifoldCF)    |

### Error Response Format
Use Tika server's error format (plain text body with status code).

---

## Request Buffering & Temp Files

### Thresholds & Configuration
- **Memory Buffer Limit**: 50 MB (configurable via `BUFFER_THRESHOLD_BYTES`)
- **Temp Directory**: Configurable via `TEMP_DIR` (default: `/tmp/tika-proxy`)
- **Cleanup Strategy**:
  1. On startup: delete all files in `TEMP_DIR` older than TTL
  2. Per-request: after response is sent, mark temp file for deletion
  3. Background: periodic cleanup of files exceeding TTL (configurable: `TEMP_FILE_TTL_MINUTES`, default 360 = 6 hours)

### Buffering Flow
1. Read request body into memory
2. If size exceeds `BUFFER_THRESHOLD_BYTES`: spill to temp file
3. Keep reference to buffered data (memory or file path)
4. Reuse for all backend calls (no re-reading from request)
5. Clean up temp file after response is returned

---

## Metadata & Text Normalization

### Text Normalization (for `/tika`)
- Normalize line endings to `\n` (convert `\r\n` and `\r` to `\n`)
- Ensure UTF-8 encoding (convert or substitute invalid UTF-8 sequences)
- Return as `text/plain; charset=utf-8`

### Metadata Normalization (for `/meta`)
**Phase 1 (current):**
- Return backend response as-is (Docling or Tika)
- Always include proxy-added fields:
  - `parser_used`: backend name ("docling" or "tika")
  - `Content-Type`: detected MIME type
  - `X-Request-ID`: UUID of request
  - `X-Fallback-Used`: boolean, true if fallback to secondary backend occurred

**Phase 2 (future work):**
- Implement proper flattening (nested → dot-notation)
- Normalize conflicting fields across backends
- Canonicalize date/time formats, author names, etc.

### Response Format
Return as `application/json`

---

## Health Endpoints

### `GET /healthz`
**Purpose**: Process-level liveness check
**Response**:
- **200 OK**: Proxy process is alive and responsive
- **500 Internal Server Error**: Proxy crashed or unresponsive
**Behavior**: Does NOT check backend availability; no external calls

### `GET /readyz`
**Purpose**: Readiness check (backend availability)
**Response**:
- **200 OK**: Both backends are reachable and responding
- **200 OK with warning**: One backend unavailable; one is healthy (or in degraded state)
- **503 Service Unavailable**: Both backends unreachable or unhealthy
**Checks**:
- HTTP HEAD or GET to each backend's health endpoint (if available)
- Fallback: attempt lightweight request (e.g., `PUT /detect/stream` with empty body or test data)
- Timeout: same as configured backend timeouts

**Response Body** (optional): JSON object
```json
{
  "status": "ready",
  "docling": { "status": "ok", "latency_ms": 15 },
  "tika": { "status": "warning", "error": "slow response" },
  "checks_timestamp": "2026-04-17T10:30:45Z"
}
```

---

## Configuration via Environment Variables

| Variable                     | Type   | Default           | Description                                                  |
| ---------------------------- | ------ | ----------------- | ------------------------------------------------------------ |
| `DOCLING_SERVICE_URL`        | string | (required)        | HTTP endpoint of Docling Serve (e.g., `http://docling:5000`) |
| `DOCLING_SERVICE_TIMEOUT_MS` | int    | 30000             | Request timeout for Docling in milliseconds                  |
| `TIKA_SERVICE_URL`           | string | (required)        | HTTP endpoint of Tika Server (e.g., `http://tika:9998`)      |
| `TIKA_SERVICE_TIMEOUT_MS`    | int    | 30000             | Request timeout for Tika in milliseconds                     |
| `BUFFER_THRESHOLD_BYTES`     | int    | 52428800          | Threshold for spooling to temp file (50 MB default)          |
| `TEMP_DIR`                   | string | `/tmp/tika-proxy` | Directory for temporary files                                |
| `TEMP_FILE_TTL_MINUTES`      | int    | 360               | Cleanup TTL for temp files in minutes (6 hours default)      |
| `PORT`                       | int    | 5000              | Listen port for proxy                                        |
| `LOG_LEVEL`                  | string | `INFO`            | Log level (DEBUG, INFO, WARN, ERROR)                         |

---

## Observability

### Request IDs
- Generate UUID v4 for each request (assigned before buffering)
- Include in all logs and response headers (`X-Request-ID`)

### Metrics (Prometheus format)
- `tika_proxy_requests_total{method, endpoint, backend, status_code}`: counter
- `tika_proxy_request_duration_seconds{method, endpoint, backend}`: histogram
- `tika_proxy_backend_fallback_total{from, to}`: counter (Docling→Tika, Tika→Docling)
- `tika_proxy_backend_selection{endpoint, backend}`: gauge (which backend chosen)
- `tika_proxy_buffer_spill_to_disk_total`: counter (times buffered file spilled to disk)
- `tika_proxy_healthcheck_failures{backend}`: counter

### Logs (structured JSON)
- All logs: JSON format with fields: `timestamp`, `level`, `request_id`, `message`, `backend`, `duration_ms`, `status_code`
- Examples: request received, backend called, fallback triggered, error occurred

---

## Future Work

### Immediate (Next Phase)
- [ ] Timeout handling refinement (granular per-backend, circuit breaker)
- [ ] Input validation (Content-Length limits, reject empty bodies, validate headers)
- [ ] Retry logic (configurable retries per backend, exponential backoff)
- [ ] MIME type detection caching (by content hash or signature)

### Medium Term
- [ ] Advanced metadata normalization (flattening, field canonicalization)
- [ ] Conflict resolution across backends (which source to trust for overlapping metadata)
- [ ] Support for custom extraction profiles (e.g., OCR tuning, language settings)
- [ ] Response streaming (for very large extracted texts)

### Long Term
- [ ] Embedded vector generation (light wrapper around embedding services)
- [ ] Custom ManifoldCF connector (if HTTP proxy becomes bottleneck)
- [ ] Multi-region deployment (geo-distribution, failover)

---

## Testing Strategy

### Unit Tests
- Routing logic (extension → backend selection)
- Normalization (line endings, UTF-8, JSON structure)
- Status code mapping (input errors → correct HTTP status)
- Buffering (in-memory vs. temp file, cleanup)

### Integration Tests
- End-to-end: `PUT /meta`, `PUT /tika`, `PUT /detect/stream` with real backends
- Fallback behavior: Docling failure → Tika success
- Health endpoints: `/healthz` and `/readyz` with backends up/down
- Error cases: backend timeout (should return 503), unsupported format (422)

### Manual Testing
- ManifoldCF integration test (actual crawl with proxy)
- Sample documents: PDFs, DOCX, images, legacy formats, encrypted files

---

## Deployment

### Docker
- **Dockerfile**: Multi-stage build, minimal final image
- **Base Image**: Python 3.11+ or Node.js depending on language choice
- **docker-compose.yml**: Services for proxy + Docling + Tika with volume mounts for temp dir

### Environment Setup
- Proxy listens on configurable `PORT` (default 5000)
- Backend service URLs via environment variables (see Configuration section)
- Temp directory mounted as volume or use ephemeral storage
- Health check configured in docker-compose (periodically call `/healthz`)
