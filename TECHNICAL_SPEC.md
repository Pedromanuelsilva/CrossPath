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
- **Default Connector Request Signals**: The stock ManifoldCF `tikaservice` connector may send `Content-Type` when `RepositoryDocument.getMimeType()` is available. It does not send a filename header by default. Filename-based routing must therefore be treated as an optional enhancement for deployments that add `Content-Disposition` or a configured trusted filename header.

### Backend Services
- **Docling**: onprem container (docling-serve) — HTTP endpoint, no authentication
- **Tika**: onprem container based on Apache Tika Server, using a custom image/configuration as needed; when OCR support is required, the image should include the necessary OCR dependencies — HTTP endpoint, no authentication

---

## Routing Logic

### Request Type Determination
The proxy determines which backend(s) or extractor pipeline(s) to call based on request path, MIME type, file extension, and document-specific routing rules.

For stock ManifoldCF `tikaservice` compatibility, generic route selection must prefer a reliable inbound `Content-Type` header when present. If no usable MIME type is present, CrossPath should use a reliable filename/extension signal when available. If neither signal is available, the proxy must default to Tika-first handling rather than guessing a Docling-preferred route.

### Generic Routing Signal Resolution
For `PUT /meta` and `PUT /tika`, CrossPath should resolve generic routing signals in a deterministic order.

Recommended initial order:
1. `Content-Type` header, if present and mapped to a known route
2. `Content-Disposition` filename parameter, if present and parseable
3. an explicitly configured trusted filename header if the deployment adds one upstream
4. no generic routing signal

Rules:
- treat `Content-Type` as the default stock-ManifoldCF-compatible routing signal
- normalize MIME type by lowercasing and ignoring parameters such as `charset`
- do not reject otherwise valid extraction requests only because `Content-Type` is missing
- normalize the candidate filename to its basename before extracting the extension
- treat the extension as case-insensitive and normalize it to lowercase
- use only the final suffix for extension routing (for example, `report.final.PDF` resolves to `.pdf`)
- do not fabricate a filename or extension from `Content-Type`
- do not guess the extension from MIME detection before primary backend selection for `/meta` or `/tika`
- if MIME type is missing or unrecognized and filename signal is missing, malformed, empty, or not trusted, treat the request as having no reliable generic routing signal

Examples:
- `Content-Type: application/pdf` → PDF route
- `Content-Type: application/vnd.openxmlformats-officedocument.wordprocessingml.document` → DOCX route
- `Content-Disposition: attachment; filename="Quarterly Report.PDF"` → `.pdf`
- trusted upstream filename header with value `exports/table.CSV` → `.csv`
- no `Content-Type`, no filename metadata → no reliable generic routing signal, so route Tika-first

The goal is to keep generic routing predictable and compatible with the default ManifoldCF external Tika Server connector. MIME detection and lightweight content inspection may still be used for document-specific routing, but they should not be used to fabricate a generic filename or extension when the request did not provide one.

### Document-Specific Routing
CrossPath must support routing for known document patterns where generic extraction is insufficient.

Examples include:
- PDFs with a known recurring business structure
- XLSX or CSV exports with well-defined layouts
- organization-specific templates that benefit from custom parsing logic

For these cases, CrossPath should:
- detect the document pattern using filename rules, metadata hints, MIME type, and/or lightweight content inspection
- route the document to a custom extractor or normalization pipeline when a matching rule is found
- fall back to the standard MIME/extension-based Docling or Tika path when no custom rule matches or when the custom extractor fails
- preserve outward Tika-compatible behavior on `/meta` and `/tika`

Document-specific routing is allowed to use richer signals than generic routing. In other words, CrossPath may use filename patterns, MIME hints, or leading-byte inspection to match a known custom extractor rule, while still refusing to invent a generic filename or extension when the request did not provide one.

### Rule Definition Model
Document-specific routing rules should be defined in code.

The intended model is:
- a lightweight document classification or detection function that inspects the MIME type, file extension, filename, and selected content/header bytes
- a code-defined registry or dictionary of matchers and routing targets
- deterministic routing decisions based on the first matching rule or explicit rule priority

### Backend Abstraction Model
CrossPath should isolate parser/backend integration from endpoint handling so that adding a new parser does not require structural changes to the HTTP layer.

The intended model is:
- a small backend adapter interface implemented by every parser backend (for example: `extract_text`, `extract_metadata`, `detect_mime` when supported, `health_check`)
- a backend registry keyed by stable backend names such as `docling`, `tika`, or future parsers
- routing policies that refer only to backend names, not concrete client classes inside endpoint code
- a fallback executor that accepts a resolved ordered backend plan and runs it without knowing parser-specific details

With this structure:
- adding a new parser means implementing one new adapter and registering it
- changing a primary backend or fallback chain means editing routing policy data, not endpoint flow
- endpoint handlers keep the same orchestration structure regardless of how many parsers exist

CrossPath should avoid hardcoding `if backend == docling` style branching throughout the request path except where a backend-specific capability is genuinely unique.

Example pattern:
- if file type is PDF and the extracted header or leading content starts with `Relatorio de Contas`, use a custom processor
- otherwise continue through the normal Docling or Tika pipeline

This should remain lightweight and request-scoped. CrossPath should not require a database or external rule store for document-specific routing.

**For `PUT /detect/stream`:**
- Always call Tika Server for MIME type detection (primary)

**For `PUT /meta` and `PUT /tika`:**

| Signal | Primary | Fallback |
| ------ | ------- | -------- |
| MIME `application/pdf` or extension `.pdf` | Docling | Tika |
| MIME `application/vnd.openxmlformats-officedocument.wordprocessingml.document` or extension `.docx` | Docling | Tika |
| MIME `application/vnd.openxmlformats-officedocument.presentationml.presentation` or extension `.pptx` | Docling | Tika |
| MIME `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` or extension `.xlsx` | Docling | Tika |
| MIME `text/csv`, `application/csv`, or extension `.csv` | Docling | Tika |
| MIME `text/markdown`, `text/x-markdown`, or extension `.md` | Docling | Tika |
| MIME `text/html`, `application/xhtml+xml`, or extensions `.html`, `.xhtml` | Docling | Tika |
| MIME `application/msword` or extension `.doc` | Tika | none |
| MIME `application/vnd.ms-powerpoint` or extension `.ppt` | Tika | none |
| MIME `application/vnd.ms-excel` or extension `.xls` | Tika | none |
| MIME `application/rtf`, `text/rtf`, or extension `.rtf` | Tika | none |
| MIME `application/vnd.ms-outlook` or extension `.msg` | Tika | none |
| all other or missing signals | Tika | none |

### Fallback Behavior
1. **Docling primary formats**: if Docling does not support the file or fails, route to Tika.
2. **Tika primary formats and unknown formats**: do not fall back to Docling.
3. **Custom extractor routes**: if a custom extractor is selected and fails, fall back according to the configured rule for that document type; this may mean retrying with Docling or Tika.
4. **No backend available**: return appropriate Tika-compatible error status (typically 422 or 500 depending on failure mode).
5. **One backend unavailable** (during /readyz): log and flag; other backend takes requests.
6. **Both backends unavailable**: return an endpoint-compatible failure for extraction requests and an unhealthy status for readiness checks.

Fallback chains should be represented as ordered backend lists or equivalent route policy objects rather than single-purpose conditional code. Even if the initial implementation only uses primary plus one fallback, the internal representation should support more than two backends so future parser additions do not require refactoring the routing structure.

### Fallback Rationale
The routing table reflects Docling's supported format set. Every format that Docling supports is routed Docling-first with Tika as fallback. Formats not supported by Docling route directly to Tika with no fallback.

Rationale:
- Docling supports a more limited number of file formats than Tika; the routing table encodes exactly which formats Docling can handle
- All Docling-supported formats use Docling as primary and Tika as fallback, so Tika always catches failures or unsupported edge cases for those formats
- Formats outside Docling's supported set route directly to Tika; there is no value in attempting Docling for formats it cannot process
- This approach means the fallback direction is always Docling → Tika, never Tika → Docling, which keeps failure behavior simple and avoids speculative second-pass parsing

If Docling adds support for additional formats, those formats should be moved from Tika-only to Docling-first in the routing table.

### Circuit Breaker Behavior
CrossPath should implement an in-memory circuit breaker per backend and per process.

Purpose:
- avoid repeatedly calling a backend that is known to be failing
- reduce timeout amplification and worker exhaustion
- improve fallback latency when one backend is temporarily unhealthy

Behavior:
- Track consecutive failures and/or timeout-driven failures per backend
- Open the circuit when the configured failure threshold is reached
- While open, skip calls to the failing backend and route directly to the configured fallback path when one exists
- After a cooldown window, transition to half-open and allow a limited number of probe requests
- Close the circuit again when probe requests succeed; reopen it if probe requests fail

Scope:
- circuit breaker state is local to each CrossPath process
- no Redis or external coordination is required

---

## Error Handling & HTTP Status Codes

### Status Code Mapping

| Code | Meaning               | When to Use                                                                                  |
| ---- | --------------------- | -------------------------------------------------------------------------------------------- |
| 200  | OK                    | Successful extraction with content in response body                                          |
| 204  | No Content            | Valid document processed but no extractable content (e.g., blank/image-only PDF without OCR) |
| 400  | Bad Request           | Invalid request syntax or invalid required endpoint semantics                                 |
| 413  | Payload Too Large     | Request body exceeds configured maximum size                                                 |
| 422  | Unprocessable Entity  | Document format not supported, encrypted, or rejected by backend                             |
| 500  | Internal Server Error | Proxy bug (not backend unavailability)                                                       |

### Error Response Format
Use plain text error responses and Tika-like status handling. Do not introduce a custom JSON error envelope on Tika-compatible endpoints.

Error responses should:
- use `Content-Type: text/plain`
- include `X-Request-ID`
- return a short human-readable message only
- avoid stack traces or structured JSON in the response body

Example messages:
- `400`: `Invalid request`
- `413`: `Request body exceeds maximum size`
- `422`: `Unsupported document format`
- `500`: `Backend service unavailable`

---

## Request Buffering & Temp Files

### Thresholds & Configuration
- **Memory Buffer Limit**: 50 MB (configurable via `BUFFER_THRESHOLD_BYTES`)
- **Maximum Request Size**: Configurable via `MAX_REQUEST_SIZE_BYTES`
- **Temp Directory**: Configurable via `TEMP_DIR` (default: `/tmp/tika-proxy`)
- **Cleanup Strategy**:
  1. Required: on startup, delete all files in `TEMP_DIR` older than TTL
  2. Required: per-request, delete or schedule deletion of the temp file after the response is sent
  3. Optional: background periodic cleanup of files exceeding TTL (configurable: `TEMP_FILE_TTL_MINUTES`, default 360 = 6 hours)

For the initial implementation, startup cleanup and request-scoped cleanup are mandatory. A periodic background sweeper is explicitly optional and may be deferred without changing the buffering architecture.

Oversized request handling:
- enforce `MAX_REQUEST_SIZE_BYTES` before full buffering when `Content-Length` is available
- if the request exceeds the configured maximum size, return `413 Payload Too Large`
- if `Content-Length` is absent or untrusted, stop buffering and reject once the streamed body crosses the configured limit
- log rejections with request ID and observed size when available

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

Initial mapping guidance:
- preserve Tika-native fields as-is when the Tika backend produced them
- for Docling-derived metadata, emit a flat Tika-compatible key/value structure rather than a Docling-native nested schema
- include only fields that can be mapped confidently in the first release
- when a Docling field has no safe Tika-compatible equivalent, omit it from the response body and prefer logging over inventing new response fields

Minimum first-release mapping table:

| Docling concept | Tika-compatible field |
| --------------- | --------------------- |
| filename        | `resourceName`        |
| content type / mime type | `Content-Type` |
| title           | `dc:title`            |
| author / authors | `dc:creator`         |
| language        | `dc:language`         |
| created / creation date | `dcterms:created` |
| modified date   | `dcterms:modified`    |

If Docling returns multiple authors, normalize them into the Tika-compatible representation chosen by implementation and document that exact encoding before release.

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
- **200 OK**: Service is operational for at least one valid extraction path
- **200 OK (degraded)**: One or more backends unavailable, but CrossPath can still serve requests through remaining supported routes
- **500 Internal Server Error**: No usable extraction path remains, or the proxy itself is unhealthy
**Checks**:
- Docling: use `GET /health`
- Tika: use `GET /` or `GET /tika` as a lightweight liveness/readiness check
- Avoid assuming generic `HEAD` support unless verified for the deployed backend version
- Fallback for Tika if needed: attempt lightweight request (e.g., `PUT /detect/stream` with empty body or test data)
- Timeout: same as configured backend timeouts

**Readiness Semantics**:
- If both Docling and Tika are healthy, report `ready`
- If one backend is unavailable but the remaining backend still provides at least one valid extraction path, report `degraded` with HTTP 200
- If Tika is unavailable, treat the service as degraded only if the deployment intentionally accepts loss of Tika-only formats and MIME detection; otherwise report unhealthy
- If no usable extraction path remains for the intended deployment, report `unhealthy` with HTTP 500
- The response body should make the distinction explicit so orchestrators and operators can tell the difference between degraded and unusable

**Response Body** (optional): JSON object
```json
{
  "status": "degraded",
  "docling": { "status": "ok", "latency_ms": 15 },
  "tika": { "status": "down", "error": "timeout" },
  "service_mode": "degraded_but_operational",
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
| `TIKA_SERVICE_TIMEOUT_MS`    | int    | 120000                                | Request timeout for Tika in milliseconds                        |
| `BUFFER_THRESHOLD_BYTES`     | int    | 52428800                              | Threshold for spooling to temp file (50 MB default)             |
| `MAX_REQUEST_SIZE_BYTES`     | int    | 524288000                             | Maximum accepted request body size before returning `413`       |
| `MAX_DOCLING_FILE_SIZE_BYTES` | int   | 104857600                             | Maximum file size eligible for Docling routing                  |
| `TEMP_DIR`                   | string | `/tmp/tika-proxy`                     | Directory for temporary files                                   |
| `TEMP_FILE_TTL_MINUTES`      | int    | 360                                   | Cleanup TTL for temp files in minutes (6 hours default)         |
| `PORT`                       | int    | 5000                                  | Listen port for proxy                                           |
| `WORKERS`                    | int    | 2                                     | Worker process count for Uvicorn or Gunicorn/Uvicorn deployment |
| `CIRCUIT_BREAKER_FAILURE_THRESHOLD` | int | 5 | Consecutive backend failures before opening the circuit |
| `CIRCUIT_BREAKER_OPEN_SECONDS` | int | 60 | Cooldown window before retrying a failed backend |
| `CIRCUIT_BREAKER_HALF_OPEN_MAX_PROBES` | int | 1 | Maximum probe requests allowed while half-open |
| `LOG_LEVEL`                  | string | `INFO`                                | Minimum log level/profile (`PRODUCTION`, `DEBUG`, `INFO`, `WARNING`/`WARN`, `ERROR`, `CRITICAL`) |

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
- `tika_proxy_circuit_breaker_state{backend, state}`: gauge
- `tika_proxy_circuit_breaker_open_total{backend}`: counter

### Logs (structured JSON)
- All logs: JSON format with fields: `timestamp`, `level`, `request_id`, `message`, `backend`, `duration_ms`, `status_code`
- Examples: request received, backend called, fallback triggered, error occurred
- Logging must support multiple configured levels/profiles:
  - `PRODUCTION`: production-safe logging; equivalent to an `INFO` minimum, with debug payloads, stack traces, and sensitive request/response details suppressed
  - `DEBUG`: verbose troubleshooting logs, including routing decisions, backend call details, fallback decisions, and sanitized request context
  - `INFO`: normal lifecycle events, backend selections, successful fallbacks, startup/shutdown, and health state transitions
  - `WARNING`/`WARN`: degraded but recoverable conditions, including backend unavailability with a usable fallback, circuit breaker transitions, cleanup failures, and near-limit resource usage
  - `ERROR`: failed requests, exhausted fallback chains, backend errors with no usable alternative, and unexpected recoverable proxy errors
  - `CRITICAL`: unrecoverable startup/configuration failures or conditions requiring process termination
- `WARN` should be accepted as an alias for `WARNING`.
- `PRODUCTION` should be accepted as a logging profile, not emitted as a log record severity; emitted log records should still use standard severities such as `INFO`, `WARNING`, `ERROR`, and `CRITICAL`.

---

## Future Work

### Immediate (Next Phase)
- [ ] Timeout handling refinement (granular per-backend)
- [ ] Input validation (Content-Length limits, reject empty bodies, validate headers)
- [ ] Retry logic (configurable retries per backend, exponential backoff)
- [ ] MIME type detection caching (by content hash or signature)
- [ ] Implement per-backend in-memory circuit breaker behavior

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
- Routing logic (MIME type / extension → backend selection)
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
