# CrossPath — Technical Specification

## Overview
This document provides detailed technical requirements for implementing CrossPath, a Tika-compatible proxy service.

---

## System Integration

### ManifoldCF
- **Version**: Latest (as of implementation)
- **Connector**: "External Tika server" connector (see Apache ManifoldCF official docs)
- **API Contract**: `PUT /meta`, `PUT /tika`, `PUT /detect/stream`
- **Expected Behavior**: Expects Tika-compatible behavior on `PUT /meta`, `PUT /tika`, and `PUT /detect/stream`; content negotiation and status codes should follow Apache Tika Server behavior as closely as practical

### Backend Services
- **Docling**: onprem container (docling-serve) — HTTP endpoint, no authentication
- **Tika**: onprem container based on Apache Tika Server, using a custom image/configuration as needed; when OCR support is required, the image should include the necessary OCR dependencies — HTTP endpoint, no authentication

---

## Routing Logic

### Request Type Determination
The proxy determines which backend(s) or extractor pipeline(s) to call based on request path, file extension, and document-specific routing rules.

If the incoming request does not provide a reliable filename or extension signal, the proxy must default to Tika-first handling rather than guessing a Docling-preferred route.

### Document-Specific Routing
CrossPath must support routing for known document patterns where generic extraction is insufficient.

Examples include:
- PDFs with a known recurring business structure
- XLSX or CSV exports with well-defined layouts
- organization-specific templates that benefit from custom parsing logic

For these cases, CrossPath should:
- detect the document pattern using filename rules, metadata hints, MIME type, and/or lightweight content inspection
- route the document to a custom extractor or normalization pipeline when a matching rule is found
- fall back to the standard Docling or Tika path when no custom rule matches or when the custom extractor fails
- preserve outward Tika-compatible behavior on `/meta` and `/tika`

### Rule Definition Model
Document-specific routing rules should be defined in code.

The intended model is:
- a lightweight document classification or detection function that inspects the file extension, filename, MIME hints, and selected content/header bytes
- a code-defined registry or dictionary of matchers and routing targets
- deterministic routing decisions based on the first matching rule or explicit rule priority

Example pattern:
- if file type is PDF and the extracted header or leading content starts with `Relatorio de Contas`, use a custom processor
- otherwise continue through the normal Docling or Tika pipeline

This should remain lightweight and request-scoped. CrossPath should not require a database or external rule store for document-specific routing.

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
3. **Custom extractor routes**: if a custom extractor is selected and fails, fall back according to the configured rule for that document type; this may mean retrying with Docling or Tika.
4. **No backend available**: return appropriate Tika-compatible error status (typically 422 or 500 depending on failure mode).
5. **One backend unavailable** (during /readyz): log and flag; other backend takes requests.
6. **Both backends unavailable**: return an endpoint-compatible failure for extraction requests and an unhealthy status for readiness checks.

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

### Error Response Format
Use plain text error responses and Tika-like status handling. Do not introduce a custom JSON error envelope on Tika-compatible endpoints.

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
- Default to Tika-compatible plain-text output for `/tika`
- Respect Tika-style content negotiation where practical for supported output formats
- Normalize line endings to `\n` for plain-text responses
- Ensure UTF-8 encoding for plain-text responses

### Metadata Normalization (for `/meta`)
- `/meta` must preserve Apache Tika-compatible response semantics
- Default `/meta` output should match Tika's default metadata response behavior
- When the caller requests JSON via `Accept: application/json`, Tika backend responses are passed through in their native JSON format
- Docling responses must be transformed to the same metadata structure expected from Tika so ManifoldCF sees a Tika-compatible response regardless of backend used
- Proxy-added trace fields must not change the response shape in a way that breaks Tika compatibility; if added, they should follow Tika-compatible conventions or be omitted from the response body and logged/returned via headers instead

### Response Format
Return in a Tika-compatible format for the requested endpoint and negotiated response type

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
- **500 Internal Server Error**: Both backends unreachable or unhealthy
**Checks**:
- Docling: use `GET /health`
- Tika: use `GET /` or `GET /tika` as a lightweight liveness/readiness check
- Avoid assuming generic `HEAD` support unless verified for the deployed backend version
- Fallback for Tika if needed: attempt lightweight request (e.g., `PUT /detect/stream` with empty body or test data)
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

| Variable                     | Type   | Default                               | Description                                                     |
| ---------------------------- | ------ | ------------------------------------- | --------------------------------------------------------------- |
| `DOCLING_SERVICE_URL`        | string | (required)                            | HTTP endpoint of Docling Serve (e.g., `http://docling:5001`)    |
| `DOCLING_SERVICE_TIMEOUT_MS` | int    | 30000                                 | Request timeout for Docling in milliseconds                     |
| `TIKA_SERVICE_URL`           | string | (required)                            | HTTP endpoint of Tika Server (e.g., `http://tika:9998`)         |
| `TIKA_SERVICE_TIMEOUT_MS`    | int    | 30000                                 | Request timeout for Tika in milliseconds                        |
| `BUFFER_THRESHOLD_BYTES`     | int    | 52428800                              | Threshold for spooling to temp file (50 MB default)             |
| `TEMP_DIR`                   | string | `/tmp/tika-proxy`                     | Directory for temporary files                                   |
| `TEMP_FILE_TTL_MINUTES`      | int    | 360                                   | Cleanup TTL for temp files in minutes (6 hours default)         |
| `PORT`                       | int    | 5000                                  | Listen port for proxy                                           |
| `WORKERS`                    | int    | conservative deployment-defined value | Worker process count for Uvicorn or Gunicorn/Uvicorn deployment |
| `LOG_LEVEL`                  | string | `INFO`                                | Log level (DEBUG, INFO, WARN, ERROR)                            |

---

## Observability

### Request IDs
- Generate UUID v4 for each request (assigned before buffering)
- Include in all logs and response headers (`X-Request-ID`)

### Metrics (Prometheus format)
- `tika_proxy_requests_total{method, endpoint, backend, status_code}`: counter
- `tika_proxy_request_duration_seconds{method, endpoint, backend}`: histogram
- `tika_proxy_backend_fallback_total{from, to}`: counter (Docling→Tika)
- `tika_proxy_backend_selection{endpoint, backend}`: gauge (which backend chosen)
- `tika_proxy_custom_route_total{route_name, outcome}`: counter (document-specific matcher/extractor selections and results)
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
- [ ] Implement document-specific routing rules for known business document patterns
- [ ] Support custom extractors for specific PDF, XLSX, CSV, and export formats
- [ ] Add lightweight document classification/matching before extractor selection
- [ ] Response streaming (for very large extracted texts)

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
- Custom routing behavior: known document pattern → custom extractor route
- Health endpoints: `/healthz` and `/readyz` with backends up/down
- Error cases: backend timeout/failure (should follow Tika-compatible failure behavior), unsupported format (422)

### Manual Testing
- ManifoldCF integration test (actual crawl with proxy)
- Sample documents: PDFs, DOCX, images, legacy formats, encrypted files

---

## Deployment

### Runtime & Deployment
- **Language**: Python 3.13+
- **Framework**: FastAPI
- **Server**: Uvicorn or Gunicorn with Uvicorn workers
- **Containerization**: Docker container
- **Networking**: internal network access to Docling Serve and Tika Server
- **Concurrency**:
  - use multiple worker processes
  - keep worker count bounded and conservative because document parsing is expensive and backend services may be the bottleneck
  - FastAPI deployment should support configuring workers via `--workers` to use multiple CPU cores

### Docker
- **Dockerfile**: Multi-stage build, minimal final image
- **Base Image**: Python 3.13+
- **docker-compose.yml**: Example services for proxy + Docling + Tika with volume mounts for temp dir; image tags should be configurable and Tika may use a custom image/configuration

### Environment Setup
- Proxy listens on configurable `PORT` (default 5000)
- Backend service URLs via environment variables (see Configuration section)
- Temp directory mounted as volume or use ephemeral storage
- Health check configured in docker-compose (periodically call `/healthz`)
