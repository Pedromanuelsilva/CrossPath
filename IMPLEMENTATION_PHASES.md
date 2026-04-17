# CrossPath — Implementation Phases

This document breaks down the implementation into executable phases for development agents.

---

## Phase 1: Project Setup & Core Scaffolding

**Goal**: Establish project structure, configuration, and basic framework

**Tasks**:
1. Create project directory structure
   - `src/` — main source code
   - `tests/` — unit and integration tests
   - `docker/` — Dockerfile and docker-compose.yml
   - `docs/` — additional documentation
2. Setup configuration management
   - Read environment variables (see TECHNICAL_SPEC.md, Configuration section)
   - Validate required variables (DOCLING_SERVICE_URL, TIKA_SERVICE_URL)
   - Provide sensible defaults for optional variables
3. Create logging framework
   - Structured JSON logging
   - Request ID (UUID v4) injected into all logs
   - Log level configuration
4. Create Prometheus metrics framework
   - Register all required metrics (see TECHNICAL_SPEC.md, Observability)
   - Expose `/metrics` endpoint (standard Prometheus)
5. Create HTTP request ID middleware/decorator
    - Generate UUID v4 for each request
    - Add to context/thread-local for downstream use
    - Include in response headers
6. Lock runtime choices in implementation
   - Python 3.13+
   - FastAPI application
   - Uvicorn or Gunicorn with Uvicorn workers
   - Docker-based deployment
   - Internal network access to Docling Serve and Tika Server
7. Define conservative process concurrency defaults
   - Use multiple worker processes
   - Keep worker count bounded because parsing is expensive and backends may be the bottleneck
   - Make worker count configurable at deployment time

**Dependencies**: None (foundational)

**Verification**:
- Configuration loads from environment without errors
- Logs output as valid JSON
- Request IDs are unique UUIDs
- `/metrics` endpoint is reachable

---

## Phase 2: HTTP API Endpoints & Request Handling

**Goal**: Implement the Tika-compatible HTTP API

**Tasks**:
1. Create HTTP server setup
   - Listen on configurable PORT (default 5000)
   - Graceful shutdown handler
2. Implement `PUT /meta` endpoint
   - Accept raw file bytes in request body
   - Return Tika-compatible metadata response
   - Placeholder response (no backend calls yet)
3. Implement `PUT /tika` endpoint
   - Accept raw file bytes in request body
   - Return Tika-compatible extracted-content response
   - Placeholder response (no backend calls yet)
4. Implement `PUT /detect/stream` endpoint
   - Accept raw file bytes in request body
   - Return plain text MIME type
   - Placeholder response (no backend calls yet)
5. Implement `GET /healthz` endpoint (liveness check only)
   - Return 200 if process is alive
   - No backend calls
6. Implement `GET /readyz` endpoint (readiness check)
   - Check Docling availability via `GET /health`
   - Check Tika availability via `GET /` or `GET /tika`
   - Return 200 if both OK, 200 with warning if one slow, 500 if both down
   - Populate response body with status JSON (optional but recommended)

**Dependencies**: Phase 1

**Verification**:
- All endpoints respond to requests
- Content-Type headers are correct
- `/healthz` returns 200
- `/readyz` detects backend availability

---

## Phase 3: Request Buffering & Temp File Management

**Goal**: Implement in-memory and spill-to-disk buffering for request bodies

**Tasks**:
1. Create buffer manager
   - Read request body into memory (up to BUFFER_THRESHOLD_BYTES)
   - If exceeds threshold, spill to temp file
   - Keep track of buffered data location (memory or file path)
2. Create temp file manager
   - Write to TEMP_DIR (create if not exists)
   - Generate unique temp file names (e.g., UUID-based)
   - Track creation timestamp for TTL cleanup
3. Implement buffer reuse
   - Same buffered data can be read multiple times
   - For memory: in-place
   - For files: re-open and seek
4. Implement temp file cleanup
   - On startup: scan TEMP_DIR and delete files older than TTL
   - Per-request: delete temp file after response sent
   - Background task (optional): periodic scan of TEMP_DIR every 10 min, delete expired files
5. Add metrics
   - Track buffer spill events
   - Track temp file cleanup

**Dependencies**: Phase 1

**Verification**:
- Small files (< 50MB) stay in memory
- Large files (> 50MB) spill to disk
- Same buffer can be read multiple times
- Temp files are cleaned up on startup
- Cleanup via TTL works (manually trigger or wait)

---

## Phase 4: Backend Service Clients

**Goal**: Create clients for calling Docling and Tika services

**Tasks**:
1. Create Docling client
   - Use HTTP `POST` to the official Docling Serve conversion endpoint, starting from `/v1/convert/source`
   - Wrap incoming file bytes into Docling's JSON request model rather than forwarding raw bytes directly
   - Use a file source payload that includes the filename and Base64-encoded content
   - Expect JSON response with extracted text and metadata
   - Handle timeout (DOCLING_SERVICE_TIMEOUT_MS)
   - Return normalized response object
2. Create Tika client
   - HTTP PUT requests to Tika `/meta` and `/tika` endpoints
   - Pass file content
   - Request JSON from Tika `/meta` via `Accept: application/json`
   - Parse JSON response from Tika `/meta`
   - Parse plain text response from Tika `/tika`
   - Handle timeout (TIKA_SERVICE_TIMEOUT_MS)
   - Return normalized response object
3. Create detection client (Tika only for now)
   - Call Tika `/detect/stream` endpoint
   - Return MIME type as string
4. Health check helper
   - Lightweight call to verify backend is alive
   - Used by `/readyz` endpoint

**Dependencies**: Phase 1

**Verification**:
- Docling client successfully calls Docling (test with real/mock service)
- Tika client successfully calls Tika (test with real/mock service)
- Timeouts are respected
- Errors are captured (not thrown, returned as error objects)
- Health checks work

---

## Phase 5: Routing Logic

**Goal**: Implement backend selection based on file extension

**Tasks**:
1. Create extension parser
   - Extract file extension from the inbound request context when a reliable filename or equivalent signal is available
   - Normalize to lowercase (e.g., ".PDF" → ".pdf")
   - If no reliable extension signal is available, default routing to Tika-first behavior
2. Create routing function
   - Map extension to primary backend and optional fallback backend (see TECHNICAL_SPEC.md, Routing Logic table)
   - Return (primary, fallback) tuple
   - For Docling-preferred formats, fallback is Tika
   - For Tika-only formats, fallback is none
3. For `PUT /detect/stream`
   - Always route to Tika

**Dependencies**: Phase 4

**Verification**:
- Extensions map to correct backends
- Docling-preferred formats are identified
- Tika-only formats are identified
- Unknown extensions default to Tika-only

---

## Phase 6: Normalization & Response Formatting

**Goal**: Normalize backend responses into Tika-compatible format

**Tasks**:
1. Create text normalization
   - Convert `\r\n` and `\r` to `\n`
   - Ensure UTF-8 (substitute or strip invalid sequences)
   - Return as string
2. Create metadata normalization (Phase 1 approach: minimal)
   - Ensure `/meta` preserves Apache Tika-compatible metadata response semantics, including negotiated JSON output
   - Pass Tika `/meta` responses through without schema changes for the negotiated representation
   - Transform Docling metadata into the same format expected from Tika
   - Keep proxy trace data in headers and logs unless a field can be added without breaking Tika compatibility
3. Create response formatters
   - Format `/meta` response in a Tika-compatible negotiated representation
   - Format `/tika` response in a Tika-compatible negotiated representation, defaulting to plain text
   - Format `/detect/stream` response as plain text (MIME type)
4. Create status code mapping
   - Success with content → 200
   - Valid but no output → 204
   - Unsupported/encrypted → 422
   - Processing/backend failure → 500

**Dependencies**: Phase 4, Phase 5

**Verification**:
- Text is normalized (line endings, UTF-8)
- Metadata is returned as Tika-compatible JSON without introducing incompatible proxy-specific body fields
- Status codes match expected behavior

---

## Phase 7: Fallback & Error Handling

**Goal**: Implement fallback logic and error recovery

**Tasks**:
1. Create fallback handler
   - If Docling is primary and it fails or does not support the file, call Tika
   - If Tika is primary, do not fall back to Docling
   - Log fallback events with backend names and reason
   - Increment fallback metrics
   - Return error response if backend failure occurs with no secondary available
2. Create error classification
   - Parse backend errors
   - Classify into Tika-compatible endpoint outcomes (422, 400, 500)
3. Create error response builder
   - Return appropriate status code and message (per TECHNICAL_SPEC.md, Error Handling)
   - Use plain text error responses without introducing a custom JSON error envelope
4. Implement timeout handling
   - Catch timeout exceptions from backend calls
   - Map to Tika-compatible failure behavior for the target endpoint

**Dependencies**: Phase 4, Phase 6

**Verification**:
- Primary backend failure triggers fallback
- Both backends failing returns appropriate error
- Timeout from one backend triggers fallback
- Fallback is logged and metrics updated

---

## Phase 8: Endpoint Implementation (End-to-End)

**Goal**: Wire together all components for working endpoints

**Tasks**:
1. Wire `PUT /meta` endpoint
   - Buffer request body (Phase 3)
   - Route based on extension (Phase 5)
   - Call primary backend via client (Phase 4)
   - On failure, fallback to secondary (Phase 7)
   - Normalize metadata response (Phase 6)
   - Return formatted JSON response with correct status code
2. Wire `PUT /tika` endpoint
   - Same flow as `/meta` but return text/plain
3. Wire `PUT /detect/stream` endpoint
   - Buffer request body (Phase 3)
   - Always route to Tika
   - Call Tika detect endpoint (Phase 4)
   - Return MIME type as plain text
4. Add metrics collection
   - Request count (by endpoint, backend, status)
   - Request duration
   - Backend selection
   - Fallback count

**Dependencies**: Phase 2, Phase 3, Phase 4, Phase 5, Phase 6, Phase 7

**Verification**:
- End-to-end request through endpoints works
- Correct backends are called
- Responses are formatted correctly
- Fallback is triggered on failure
- Metrics are collected

---

## Phase 9: Testing

**Goal**: Comprehensive unit and integration tests

**Tasks**:
1. Unit tests (no external services)
   - Routing logic (extension → backend selection)
   - Normalization (text, metadata, status codes)
   - Buffering (in-memory, spill, reuse)
   - Temp file cleanup (TTL, startup)
2. Integration tests (with mock/real backends)
   - `PUT /meta` with sample documents
   - `PUT /tika` with sample documents
   - `PUT /detect/stream` with sample documents
   - Fallback behavior (Docling fails → Tika succeeds)
   - Error cases (unsupported format → 422, timeout/failure → Tika-compatible failure behavior)
   - Health endpoints (`/healthz`, `/readyz`)
3. Manual testing
   - Full docker-compose with proxy, Docling, Tika running
   - Call endpoints with various document types
   - Verify outputs match expectations

**Dependencies**: Phase 8

**Verification**:
- All unit tests pass
- All integration tests pass
- Manual testing shows correct behavior

---

## Phase 10: Deployment & Documentation

**Goal**: Docker image and usage documentation

**Tasks**:
1. Create Dockerfile
   - Multi-stage build (minimal final size)
   - Install dependencies (requests, prometheus-client, etc.)
   - Copy source code
   - Expose PORT (default 5000)
   - Health check (call `/healthz`)
   - Entry point
2. Create docker-compose.yml
   - Service for proxy
   - Service for Docling Serve
   - Service for Tika Server with OCR
   - Volume for temp directory
   - Environment variable setup
   - Network isolation
3. Create README
   - Quick start with docker-compose
   - Configuration reference (all env variables)
   - API reference (endpoints, request/response examples)
   - Monitoring & troubleshooting guide
   - Future work / roadmap
4. Create startup script (if needed)
   - Source env file
   - Validate configuration
   - Start server

**Dependencies**: Phase 9 (tests passing)

**Verification**:
- Docker image builds successfully
- docker-compose runs all services
- Proxy responds to requests
- Temp directory is writable and persists between restarts

---

## Implementation Notes

### Suggested Tech Stack (not prescriptive)
- **Language**: Python 3.13+
- **Framework**: FastAPI
- **Server**: Uvicorn or Gunicorn with Uvicorn workers
- **HTTP**: Python HTTP client appropriate for FastAPI service integration
- **Metrics**: prometheus-client
- **Logging**: structured JSON logging for Python
- **Testing**: pytest

### Parallelization
- Phases 1–2 can start immediately (scaffolding)
- Phases 3–5 can run in parallel (independent subsystems)
- Phase 6–7 can start once Phase 4 is stable
- Phase 8 requires all prior phases
- Phase 9–10 start after Phase 8

### Rollout
1. Complete Phase 1–2 (API structure)
2. Complete Phase 3–5 (core infrastructure)
3. Complete Phase 6–7 (transformation logic)
4. Complete Phase 8 (integration)
5. Complete Phase 9 (testing, validation)
6. Deploy (Phase 10)
