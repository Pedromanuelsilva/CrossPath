# CrossPath Design Review — Recommended Annotations

This document contains recommended annotations and clarifications for the existing architectural documentation based on design analysis.

---

## CRITICAL — Must Address Before Implementation

### 1. TECHNICAL_SPEC.md — Docling API Integration (Line 134-138)

**Current Text:**
> Use HTTP `POST` to the official Docling Serve conversion endpoint, starting from `/v1/convert/source`
> Wrap incoming file bytes into Docling's JSON request model rather than forwarding raw bytes directly
> Use a file source payload that includes the filename and Base64-encoded content

**Issue:**
Base64 encoding adds 33% overhead to file size. A 50MB PDF becomes 66MB+ JSON payload, conflicting with the buffering strategy and causing memory/timeout issues on large documents.

**Recommended Annotation:**
```
⚠️ VERIFY: Confirm Docling Serve API supports Base64 approach for files >50MB
- Test with 100MB+ documents to validate memory usage and timeout behavior
- Alternative: Check if Docling supports multipart/form-data or binary PUT
- Consider: Size-based routing (files >100MB skip Docling, go direct to Tika)
- Document: Maximum practical file size for Docling route
```

---

### 2. TECHNICAL_SPEC.md — Routing Signal Detection (Line 24-50)

**Current Text:**
> Extract file extension from the inbound request context when a reliable filename or equivalent signal is available
> If no reliable extension signal is available, default routing to Tika-first behavior

**Issue:**
Tika API endpoints (`PUT /meta`, `/tika`) accept raw bytes with no filename in URL path. The stock ManifoldCF `tikaservice` external Tika Server connector may send `Content-Type` when `RepositoryDocument.getMimeType()` is available, but it does not send a filename header by default.

**Resolved Requirement:**
```
✅ RESOLVED: Route using MIME type first, then filename extension
- Generic route selection first uses inbound Content-Type when it maps to a known route
- If Content-Type is missing or unrecognized, route from Content-Disposition or a configured trusted filename header when available
- If neither signal exists, default to Tika-first handling
- Missing Content-Type alone must not reject a request
- Do not fabricate a filename or extension from MIME detection
```

---

### 3. TECHNICAL_SPEC.md — Fallback Asymmetry (Line 80-83)

**Current Text:**
> Docling primary formats: if Docling does not support the file or fails, route to Tika.
> Tika primary formats and unknown formats: do not fall back to Docling.

**Issue:**
If Tika times out on a `.pdf` (Docling-preferred format), no retry with Docling occurs even though Docling might succeed.

**Recommended Annotation:**
```
⚠️ DESIGN DECISION: Document rationale for asymmetric fallback
- Scenario: Tika timeout on complex PDF → 500 error (no Docling retry)
- Question: Should Docling-preferred formats have symmetric fallback?
- Tradeoff: Symmetric fallback = better reliability but 2x latency on failures
- Recommendation: Add config flag ENABLE_SYMMETRIC_FALLBACK (default: false)
- Document: Why asymmetry is intentional (if it is)
```

---

### 4. TECHNICAL_SPEC.md — Metadata Normalization (Line 135-138)

**Current Text:**
> Transform Docling metadata into the same format expected from Tika
> Pass Tika `/meta` responses through without schema changes for the negotiated representation

**Issue:**
No field mapping table provided. Docling and Tika have different metadata schemas.

**Recommended Annotation:**
```
📋 SPECIFICATION NEEDED: Define exact metadata field mapping
- Create table: Docling field → Tika field (e.g., author → dc:creator)
- Identify: Which fields are required by ManifoldCF for indexing?
- Strategy: Pass-through all fields + best-effort normalization, or strict mapping?
- Test: Compare Docling output vs Tika output for same document
- Document: Mapping table in TECHNICAL_SPEC.md before Phase 6
```

---

## HIGH PRIORITY — Address in Phase 1-3

### 5. TECHNICAL_SPEC.md — Missing Max Request Size (Configuration Section)

**Current Text:**
(No mention of maximum request size limit)

**Issue:**
Proxy could accept 5GB file and crash. No upper bound specified.

**Recommended Addition:**
```
Add to Configuration table:

| Variable                  | Type    | Default    | Description                                    |
|---------------------------|---------|------------|------------------------------------------------|
| MAX_REQUEST_SIZE_BYTES    | int     | 524288000  | Maximum request body size (500 MB default)     |
| REJECT_OVERSIZED_REQUESTS | boolean | true       | Return 413 for requests exceeding max size     |

⚠️ IMPLEMENTATION NOTE:
- Check Content-Length header before buffering
- Return 413 Payload Too Large if exceeded
- Log rejected requests with size and request ID
```

---

### 6. ENVIRONMENT_VARIABLES.md — WORKERS Default Value (Line 98-105)

**Current Text:**
> Default: conservative deployment-defined value
> Should be set conservatively because document parsing is expensive

**Issue:**
No concrete guidance on what "conservative" means. Operators won't know how to set this.

**Recommended Annotation:**
```
📊 PROVIDE CONCRETE GUIDANCE:

Default: 2
Recommended formula: min(CPU_cores / 2, 4)
Rationale: Backend services (Docling/Tika) are the bottleneck, not proxy CPU

Scaling guidance:
- 1-2 workers: Development, low-volume (<10 docs/min)
- 2-4 workers: Production, moderate volume (10-50 docs/min)
- 4+ workers: Only if backend services are scaled horizontally
- Do NOT set to CPU count — parsing is I/O bound, not CPU bound

Monitor: If request queue depth grows, scale backends first, then workers
```

---

### 7. TECHNICAL_SPEC.md — Temp File Cleanup (Line 112-114)

**Current Text:**
> Background task (optional): periodic scan of TEMP_DIR every 10 min, delete expired files

**Issue:**
Background cleanup is marked "optional" but is critical to prevent disk exhaustion.

**Recommended Change:**
```
CHANGE "optional" to "required"

Background task (required): periodic scan of TEMP_DIR every 10 min, delete expired files

⚠️ CRITICAL: Without background cleanup, disk will fill if:
- Process crashes mid-request (per-request cleanup doesn't run)
- High request volume with large files
- TTL is long (default 6 hours)

Add metric: tika_proxy_temp_dir_disk_usage_bytes (gauge)
Add alert: Disk usage >80% of available space
```

---

### 8. TECHNICAL_SPEC.md — Timeout Defaults vs OCR (Line 184-186)

**Current Text:**
> DOCLING_SERVICE_TIMEOUT_MS: Default 30000 (30 seconds)
> TIKA_SERVICE_TIMEOUT_MS: Default 30000 (30 seconds)

**Issue:**
Spec mentions OCR support in Tika image, but OCR can take 60-120s. Default timeout will cause failures.

**Recommended Annotation:**
```
⚠️ OCR CONSIDERATION:
Default 30s timeout assumes NO OCR processing.

If Tika image includes OCR (Tesseract):
- Scanned PDFs can take 60-120s per page
- Multi-page scanned documents may exceed timeout
- Recommendation: Set TIKA_SERVICE_TIMEOUT_MS=120000 (2 min) for OCR workloads

Alternative: Add separate timeout config
- TIKA_SERVICE_TIMEOUT_MS=30000 (standard documents)
- TIKA_OCR_TIMEOUT_MS=120000 (scanned/image documents)
- Route based on MIME type (image/pdf vs application/pdf)

Document: OCR timeout requirements in deployment section
```

---

## MEDIUM PRIORITY — Address in Phase 5-8

### 9. TECHNICAL_SPEC.md — Custom Extractor Interface (Line 36-56)

**Current Text:**
> Lightweight document classification or detection function
> Code-defined registry or dictionary of matchers and routing targets

**Issue:**
No concrete interface or example provided. Phase 5 implementer will need to make design decisions.

**Recommended Addition:**
```
📝 ADD EXAMPLE CUSTOM EXTRACTOR INTERFACE:

```python
class DocumentMatcher:
    def matches(self, filename: str, mime_type: str, header_bytes: bytes) -> bool:
        """Return True if this matcher handles the document"""
        pass

class CustomExtractor:
    def extract_text(self, file_bytes: bytes) -> str:
        """Extract text content"""
        pass
    
    def extract_metadata(self, file_bytes: bytes) -> dict:
        """Extract metadata in Tika-compatible format"""
        pass

# Registry example
CUSTOM_ROUTES = [
    {
        "matcher": lambda f, m, h: f.endswith(".pdf") and b"Relatorio de Contas" in h[:1024],
        "extractor": RelatorioExtractor(),
        "fallback": "docling"  # or "tika"
    }
]
```

Document: Custom extractor must return Tika-compatible format
```

---

### 10. IMPLEMENTATION_PHASES.md — Phase 4 Docling Client (Line 133-139)

**Current Text:**
> Use HTTP `POST` to the official Docling Serve conversion endpoint
> Wrap incoming file bytes into Docling's JSON request model
> Use a file source payload that includes the filename and Base64-encoded content

**Issue:**
Same as annotation #1 — Base64 encoding inefficiency.

**Recommended Annotation:**
```
⚠️ BLOCKED: Do not implement Phase 4 Docling client until annotation #1 is resolved

Before implementation:
1. Research Docling Serve API documentation for binary upload support
2. Test Base64 approach with 100MB+ files to measure memory/latency impact
3. If Base64 is required, add MAX_DOCLING_FILE_SIZE_BYTES config (default: 100MB)
4. Document: Files exceeding limit skip Docling, route direct to Tika

Alternative implementation if Base64 is problematic:
- Use multipart/form-data if supported
- Stream file from temp disk location if Docling supports file path input
- Consider Docling as "small file optimizer" only (<50MB)
```

---

### 11. TECHNICAL_SPEC.md — Add Circuit Breaker Pattern (Future Work Section)

**Current Text:**
(Not mentioned in Future Work)

**Recommended Addition:**
```
Add to "Immediate (Next Phase)" section:

- [ ] Circuit breaker for failing backends
  - Track consecutive failures per backend (threshold: 5 failures)
  - Open circuit: skip backend for cooldown period (default: 60s)
  - Half-open: test with single request after cooldown
  - Closed: resume normal routing after successful test
  - Metric: tika_proxy_circuit_breaker_state{backend, state=open|closed|half_open}
  - Rationale: Avoid 30s timeout on every request when backend is known to be down
```

---

### 12. TECHNICAL_SPEC.md — /readyz Ambiguity (Line 156-160)

**Current Text:**
> 200 OK: Both backends are reachable and responding
> 200 OK with warning: One backend unavailable; one is healthy
> 500 Internal Server Error: Both backends unreachable or unhealthy

**Issue:**
If Docling is down but all requests are Tika-only formats, proxy is fully functional but returns "warning".

**Recommended Annotation:**
```
🤔 DESIGN CLARIFICATION NEEDED:

Scenario: Docling down, but current workload is 100% Tika-only formats (.doc, .xls, .msg)
- Current spec: /readyz returns 200 with warning
- Question: Is this "degraded" or "fully operational"?

Recommendation: Add /readyz?strict=true parameter
- /readyz (default): 200 if at least one backend healthy (current behavior)
- /readyz?strict=true: 200 only if both backends healthy
- Use case: Kubernetes readiness probe uses default, monitoring uses strict

Alternative: Separate endpoints
- /readyz/docling → 200 if Docling healthy
- /readyz/tika → 200 if Tika healthy
- /readyz → 200 if either healthy
```

---

## LOW PRIORITY — Document for Operations

### 13. README.md — Add Multi-Instance Deployment Section

**Recommended Addition:**
```
## High Availability Deployment

CrossPath is stateless and can be deployed in multiple instances behind a load balancer.

Recommended setup:
- 2-3 proxy instances for redundancy
- Load balancer: Round-robin or least-connections
- Shared temp directory: Use network volume or separate temp dirs per instance
- Health checks: /healthz for liveness, /readyz for readiness

Example docker-compose with multiple instances:
```yaml
services:
  proxy-1:
    image: crosspath:latest
    environment:
      TEMP_DIR: /tmp/tika-proxy-1
    # ... other config
  
  proxy-2:
    image: crosspath:latest
    environment:
      TEMP_DIR: /tmp/tika-proxy-2
    # ... other config
  
  load-balancer:
    image: nginx:latest
    # ... nginx config for upstream proxies
```

Scaling considerations:
- Scale proxy instances based on request queue depth
- Scale backend services (Docling/Tika) based on CPU/memory usage
- Proxy is typically not the bottleneck — backends are
```

---

### 14. TECHNICAL_SPEC.md — Error Response Format (Line 101-102)

**Current Text:**
> Use plain text error responses and Tika-like status handling

**Issue:**
Exact format not specified. Just the error message? Status code in body?

**Recommended Annotation:**
```
📋 SPECIFY EXACT ERROR FORMAT:

Error response format (plain text):
```
HTTP/1.1 422 Unprocessable Entity
Content-Type: text/plain
X-Request-ID: 550e8400-e29b-41d4-a716-446655440000

Unsupported document format: encrypted PDF
```

Do NOT use JSON error envelope on Tika-compatible endpoints.
Do NOT include stack traces in production (log them instead).
DO include request ID in response header for traceability.

Example error messages:
- 400: "Invalid request"
- 413: "Request body exceeds maximum size (500 MB)"
- 422: "Unsupported document format: encrypted PDF"
- 422: "Document rejected by backend: password protected"
- 500: "Backend service unavailable"
- 500: "Request timeout after 30000ms"
```

---

### 15. TECHNICAL_SPEC.md — Graceful Shutdown (Not Mentioned)

**Recommended Addition:**
```
Add new section after "Deployment":

## Graceful Shutdown

When proxy receives SIGTERM (e.g., during container restart):
1. Stop accepting new requests (return 503 Service Unavailable)
2. Wait for in-flight requests to complete (timeout: 60s)
3. Clean up temp files for completed requests
4. Close backend connections
5. Exit with code 0

Configuration:
- SHUTDOWN_TIMEOUT_SECONDS: Default 60
- Kubernetes: Set terminationGracePeriodSeconds to match + 10s buffer

Implementation:
- FastAPI: Use lifespan context manager
- Uvicorn: Handle SIGTERM signal
- Log: "Shutting down gracefully, waiting for N in-flight requests"
```

---

## Summary of Annotations by Priority

### Must Fix Before Implementation (4 items)
1. Docling Base64 encoding inefficiency
2. Extension detection mechanism unclear
3. Fallback asymmetry rationale
4. Metadata field mapping undefined

### High Priority — Phase 1-3 (4 items)
5. Add max request size limit
6. Provide concrete WORKERS guidance
7. Make background cleanup mandatory
8. Document OCR timeout requirements

### Medium Priority — Phase 5-8 (4 items)
9. Define custom extractor interface
10. Block Phase 4 until Docling API verified
11. Add circuit breaker pattern
12. Clarify /readyz behavior

### Low Priority — Documentation (3 items)
13. Multi-instance deployment guide
14. Specify exact error response format
15. Add graceful shutdown specification

---

## Recommended Action Plan

**Before starting Phase 1:**
1. Confirm ManifoldCF tikaservice connector behavior in the target release if upgrading beyond 2.25 (annotation #2)
2. Test Docling Serve API with large files (annotation #1)
3. Create metadata field mapping table (annotation #4)

**During Phase 1:**
4. Add MAX_REQUEST_SIZE_BYTES config (annotation #5)
5. Set WORKERS default to 2 with scaling guidance (annotation #6)
6. Make background cleanup required (annotation #7)

**During Phase 4-5:**
7. Implement Docling client based on annotation #1 resolution
8. Define custom extractor interface (annotation #9)
9. Document fallback rationale (annotation #3)

**During Phase 9 (Testing):**
10. Test OCR timeout scenarios (annotation #8)
11. Validate error response formats (annotation #14)

**During Phase 10 (Deployment):**
12. Add HA deployment guide (annotation #13)
13. Document graceful shutdown (annotation #15)
14. Consider circuit breaker for v1.1 (annotation #11)
