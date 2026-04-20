# CrossPath — Environment Variables Reference

Complete reference for all environment variables and their usage.

---

## Required Variables

These must be set or the proxy will not start.

### DOCLING_SERVICE_URL
- **Type**: String (URL)
- **Example**: `http://docling:5001` or `http://192.168.1.10:5001`
- **Description**: HTTP endpoint of Docling Serve container
- **Used by**: Docling client for routing documents to Docling
- **Error handling**: Proxy fails to start if not set or invalid URL format

### TIKA_SERVICE_URL
- **Type**: String (URL)
- **Example**: `http://tika:9998` or `http://192.168.1.20:9998`
- **Description**: HTTP endpoint of Apache Tika Server container
- **Used by**: Tika client for routing documents to Tika, MIME detection
- **Error handling**: Proxy fails to start if not set or invalid URL format

---

## Optional Variables (with Defaults)

### DOCLING_SERVICE_TIMEOUT_MS
- **Type**: Integer (milliseconds)
- **Default**: `30000` (30 seconds)
- **Range**: 1000–600000 (1 second to 10 minutes)
- **Description**: HTTP request timeout when calling Docling Serve
- **Used by**: Docling client to enforce timeout on requests
- **Behavior**: If Docling does not respond within this time, request is aborted and fallback to Tika is triggered
- **Example**: `DOCLING_SERVICE_TIMEOUT_MS=45000` (45 seconds)

### TIKA_SERVICE_TIMEOUT_MS
- **Type**: Integer (milliseconds)
- **Default**: `30000` (30 seconds)
- **Range**: 1000–600000 (1 second to 10 minutes)
- **Description**: HTTP request timeout when calling Tika Server
- **Used by**: Tika client to enforce timeout on requests
- **Behavior**: If Tika does not respond within this time, request is aborted and mapped to Tika-compatible failure behavior (or fallback to secondary if applicable)
- **Example**: `TIKA_SERVICE_TIMEOUT_MS=60000` (60 seconds)
- **OCR Note**:
  - `30000` is suitable only for non-OCR or light-OCR workloads
  - If the deployed Tika image performs OCR on scanned PDFs or images, use a substantially higher timeout
  - A practical starting point for OCR-heavy deployments is `120000`

### BUFFER_THRESHOLD_BYTES
- **Type**: Integer (bytes)
- **Default**: `52428800` (50 MB)
- **Range**: 1048576–5368709120 (1 MB to 5 GB)
- **Description**: Maximum file size to keep in memory before spilling to disk
- **Used by**: Buffer manager (Phase 3)
- **Behavior**: Files smaller than this threshold are buffered in RAM; larger files are written to `TEMP_DIR`
- **Example**: `BUFFER_THRESHOLD_BYTES=104857600` (100 MB)
- **Tuning**: 
  - Increase for low-latency, high-memory systems
  - Decrease to reduce memory usage on memory-constrained systems

### MAX_REQUEST_SIZE_BYTES
- **Type**: Integer (bytes)
- **Default**: `524288000` (500 MB)
- **Range**: 1048576–21474836480 (1 MB to 20 GB)
- **Description**: Maximum accepted request body size before the proxy rejects the request
- **Used by**: Request validation and buffering guardrails
- **Behavior**:
  - If `Content-Length` exceeds this value, reject early with `413`
  - If a streamed body grows beyond this value, stop buffering and return `413`
  - This is an absolute safety limit and is separate from the in-memory spill threshold
- **Example**: `MAX_REQUEST_SIZE_BYTES=1073741824` (1 GB)

### MAX_DOCLING_FILE_SIZE_BYTES
- **Type**: Integer (bytes)
- **Default**: `104857600` (100 MB)
- **Range**: 1048576–21474836480 (1 MB to 20 GB)
- **Description**: Maximum file size eligible for Docling routing when Docling upload requires Base64-wrapped request bodies
- **Used by**: Routing policy for Docling-preferred formats
- **Behavior**:
  - Files above this threshold should skip Docling and route directly to Tika
  - This limit is separate from `MAX_REQUEST_SIZE_BYTES`
  - The purpose is to bound Base64 expansion, memory pressure, and Docling request latency
- **Example**: `MAX_DOCLING_FILE_SIZE_BYTES=157286400` (150 MB)

### TEMP_DIR
- **Type**: String (filesystem path)
- **Default**: `/tmp/tika-proxy`
- **Example Values**:
  - `/tmp/tika-proxy` (Linux/Mac)
  - `/var/tmp/tika-proxy` (alternative temp location)
  - `/home/user/tika-proxy-temp` (custom directory)
  - `C:\Windows\Temp\tika-proxy` (Windows)
- **Description**: Directory where spilled temp files are written
- **Used by**: Buffer manager and temp file manager
- **Behavior**: Directory is created if it does not exist (with appropriate permissions); must be writable by the proxy process
- **Docker**: Mount a volume to persist temp files across container restarts (e.g., `-v tika-temp:/tmp/tika-proxy`)
- **Cleanup**: On startup, all files in this directory older than `TEMP_FILE_TTL_MINUTES` are deleted

### TEMP_FILE_TTL_MINUTES
- **Type**: Integer (minutes)
- **Default**: `360` (6 hours)
- **Range**: 1–43200 (1 minute to 30 days)
- **Description**: Time-to-live for temporary files; files older than this are automatically cleaned up
- **Used by**: Temp file manager cleanup process
- **Behavior**:
  - Required: on startup, scan `TEMP_DIR` and delete files with modification time > TTL
  - Required: per-request, after response is sent, temp file is eligible for cleanup
  - Optional: periodic background scan every 10 minutes deletes expired files
  - Initial release requirement: startup cleanup and per-request cleanup are sufficient; the periodic sweeper may be added later
- **Example**: `TEMP_FILE_TTL_MINUTES=120` (2 hours)
- **Tuning**: 
  - Increase if you want longer retention for debugging
  - Decrease to keep disk usage low

### PORT
- **Type**: Integer (port number)
- **Default**: `5000`
- **Range**: 1024–65535 (non-privileged ports)
- **Description**: TCP port on which the proxy listens for HTTP requests
- **Used by**: HTTP server
- **Example**: `PORT=8080`
- **Docker**: Expose via `-p 8080:5000` in docker-compose.yml or `docker run`
- **Note**: Do not use port 80 or 443 without running as root (not recommended in containers)

### WORKERS
- **Type**: Integer (process count)
- **Default**: `2`
- **Description**: Number of FastAPI worker processes for Uvicorn or Gunicorn/Uvicorn deployment
- **Used by**: container startup command / process manager
- **Behavior**:
  - Start with `2` workers for most deployments
  - A practical heuristic is `min(CPU cores / 2, 4)`, rounded down but never less than 1
  - Prefer scaling Docling and Tika capacity before increasing proxy workers aggressively
  - Do not assume that matching CPU count is optimal; backend latency is usually the bottleneck
- **Example**: `WORKERS=2`

### CIRCUIT_BREAKER_FAILURE_THRESHOLD
- **Type**: Integer
- **Default**: `5`
- **Description**: Number of consecutive backend failures before opening the in-memory circuit breaker for that backend
- **Used by**: backend resilience logic
- **Behavior**: Once the threshold is reached, the backend is temporarily bypassed until the cooldown window expires

### CIRCUIT_BREAKER_OPEN_SECONDS
- **Type**: Integer (seconds)
- **Default**: `60`
- **Description**: Cooldown period for an open circuit before allowing half-open probe requests
- **Used by**: backend resilience logic
- **Behavior**: During this period, calls to the failing backend are skipped when a fallback route exists

### CIRCUIT_BREAKER_HALF_OPEN_MAX_PROBES
- **Type**: Integer
- **Default**: `1`
- **Description**: Maximum number of probe requests allowed while a backend circuit is half-open
- **Used by**: backend resilience logic
- **Behavior**: Successful probes close the breaker; failed probes reopen it

### LOG_LEVEL
- **Type**: String (enum)
- **Default**: `INFO`
- **Valid Values**: `DEBUG`, `INFO`, `WARN`, `ERROR`
- **Description**: Minimum log level to output
- **Used by**: Logging framework
- **Behavior**:
  - `DEBUG`: All log messages, including request/response details, backend calls
  - `INFO`: Normal operation logs, fallbacks, errors
  - `WARN`: Warnings and errors only
  - `ERROR`: Errors only (minimal logging)
- **Example**: `LOG_LEVEL=DEBUG`
- **Note**: Set to `DEBUG` in development/troubleshooting; use `INFO` or `WARN` in production

---

## Example .env File

```bash
# Backend services (REQUIRED)
DOCLING_SERVICE_URL=http://docling:5001
TIKA_SERVICE_URL=http://tika:9998

# Timeouts (optional, defaults shown)
DOCLING_SERVICE_TIMEOUT_MS=30000
TIKA_SERVICE_TIMEOUT_MS=30000

# Buffering (optional, defaults shown)
BUFFER_THRESHOLD_BYTES=52428800
MAX_REQUEST_SIZE_BYTES=524288000
MAX_DOCLING_FILE_SIZE_BYTES=104857600
TEMP_DIR=/tmp/tika-proxy
TEMP_FILE_TTL_MINUTES=360

# Proxy settings (optional, defaults shown)
PORT=5000
WORKERS=2
CIRCUIT_BREAKER_FAILURE_THRESHOLD=5
CIRCUIT_BREAKER_OPEN_SECONDS=60
CIRCUIT_BREAKER_HALF_OPEN_MAX_PROBES=1
LOG_LEVEL=INFO
```

---

## Docker Compose Usage

```yaml
services:
  proxy:
    image: crosspath:${CROSSPATH_TAG:-latest}
    ports:
      - "5000:5000"
    environment:
      DOCLING_SERVICE_URL: http://docling:5001
      TIKA_SERVICE_URL: http://tika:9998
      DOCLING_SERVICE_TIMEOUT_MS: 30000
      TIKA_SERVICE_TIMEOUT_MS: 30000
      BUFFER_THRESHOLD_BYTES: 52428800
      MAX_REQUEST_SIZE_BYTES: 524288000
      MAX_DOCLING_FILE_SIZE_BYTES: 104857600
      TEMP_DIR: /tmp/tika-proxy
      TEMP_FILE_TTL_MINUTES: 360
      PORT: 5000
      WORKERS: 2
      CIRCUIT_BREAKER_FAILURE_THRESHOLD: 5
      CIRCUIT_BREAKER_OPEN_SECONDS: 60
      CIRCUIT_BREAKER_HALF_OPEN_MAX_PROBES: 1
      LOG_LEVEL: INFO
    volumes:
      - tika-temp:/tmp/tika-proxy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/healthz"]
      interval: 10s
      timeout: 5s
      retries: 3
    depends_on:
      - docling
      - tika

  docling:
    image: ${DOCLING_IMAGE:-ghcr.io/docling-project/docling-serve:latest}
    ports:
      - "5001:5001"
    # Adjust these based on your Docling Serve image configuration

  tika:
    image: ${TIKA_IMAGE:-your-custom-tika:latest}
    ports:
      - "9998:9998"
    # Use your custom Tika image/configuration; when OCR is required, include OCR dependencies in the image

volumes:
  tika-temp:
```

---

## Validation & Error Messages

The proxy will validate configuration on startup:

| Condition | Error Message | Resolution |
|-----------|---------------|-----------|
| Missing `DOCLING_SERVICE_URL` | `DOCLING_SERVICE_URL not set` | Set `DOCLING_SERVICE_URL` in environment |
| Missing `TIKA_SERVICE_URL` | `TIKA_SERVICE_URL not set` | Set `TIKA_SERVICE_URL` in environment |
| Invalid URL format | `Invalid DOCLING_SERVICE_URL: [reason]` | Ensure URL is valid (e.g., `http://host:port`) |
| Invalid `PORT` | `PORT must be integer between 1024-65535` | Set valid port number |
| Invalid `TEMP_DIR` | `TEMP_DIR does not exist or not writable` | Create directory and set appropriate permissions |
| Invalid `BUFFER_THRESHOLD_BYTES` | `BUFFER_THRESHOLD_BYTES must be > 0` | Set to valid number of bytes |
| Invalid `MAX_REQUEST_SIZE_BYTES` | `MAX_REQUEST_SIZE_BYTES must be > 0` | Set to valid number of bytes |
| Invalid `MAX_DOCLING_FILE_SIZE_BYTES` | `MAX_DOCLING_FILE_SIZE_BYTES must be > 0` | Set to valid number of bytes |

---

## Runtime Configuration Checks

At startup, the proxy will:
1. ✅ Validate all required variables are set
2. ✅ Parse and validate URL formats
3. ✅ Validate numeric ranges (timeouts, ports, sizes)
4. ✅ Check if `TEMP_DIR` exists and is writable (create if missing)
5. ✅ Verify Docling and Tika are reachable (timeout-based health check)
6. ✅ Log all configuration values (sensitive values masked) at startup

---

## Common Configurations

### Development (Local Docker)
```bash
DOCLING_SERVICE_URL=http://docling:5001
TIKA_SERVICE_URL=http://tika:9998
DOCLING_SERVICE_TIMEOUT_MS=30000
TIKA_SERVICE_TIMEOUT_MS=30000
BUFFER_THRESHOLD_BYTES=52428800
MAX_REQUEST_SIZE_BYTES=524288000
MAX_DOCLING_FILE_SIZE_BYTES=104857600
TEMP_DIR=/tmp/tika-proxy
TEMP_FILE_TTL_MINUTES=360
PORT=5000
WORKERS=2
CIRCUIT_BREAKER_FAILURE_THRESHOLD=5
CIRCUIT_BREAKER_OPEN_SECONDS=60
CIRCUIT_BREAKER_HALF_OPEN_MAX_PROBES=1
LOG_LEVEL=DEBUG
```

### Production (longer timeouts for OCR)
```bash
DOCLING_SERVICE_URL=http://docling.internal:5001
TIKA_SERVICE_URL=http://tika.internal:9998
DOCLING_SERVICE_TIMEOUT_MS=120000
TIKA_SERVICE_TIMEOUT_MS=120000
BUFFER_THRESHOLD_BYTES=104857600
MAX_REQUEST_SIZE_BYTES=524288000
MAX_DOCLING_FILE_SIZE_BYTES=104857600
TEMP_DIR=/var/lib/tika-proxy/temp
TEMP_FILE_TTL_MINUTES=720
PORT=5000
WORKERS=2
CIRCUIT_BREAKER_FAILURE_THRESHOLD=5
CIRCUIT_BREAKER_OPEN_SECONDS=60
CIRCUIT_BREAKER_HALF_OPEN_MAX_PROBES=1
LOG_LEVEL=WARN
```

### Memory-Constrained (e.g., edge device)
```bash
DOCLING_SERVICE_URL=http://docling:5001
TIKA_SERVICE_URL=http://tika:9998
DOCLING_SERVICE_TIMEOUT_MS=60000
TIKA_SERVICE_TIMEOUT_MS=60000
BUFFER_THRESHOLD_BYTES=10485760
MAX_REQUEST_SIZE_BYTES=268435456
MAX_DOCLING_FILE_SIZE_BYTES=52428800
TEMP_DIR=/tmp/tika-proxy
TEMP_FILE_TTL_MINUTES=180
PORT=5000
WORKERS=2
CIRCUIT_BREAKER_FAILURE_THRESHOLD=5
CIRCUIT_BREAKER_OPEN_SECONDS=60
CIRCUIT_BREAKER_HALF_OPEN_MAX_PROBES=1
LOG_LEVEL=INFO
```
