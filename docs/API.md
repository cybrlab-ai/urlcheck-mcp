# URL Scanner MCP Server - API Documentation

> **Malicious URL Detection for Safe Agentic Browsing**
>
> High-accuracy MCP-native URL scanner for secure agentic browsing, powered by ML classification and browser automation.
>
> **Publisher:** [CybrLab AI](https://cybrlab.ai) | **Service:** [urlcheck.ai](https://urlcheck.ai)

---

## Table of Contents

1. [Overview](#overview)
2. [Authentication](#authentication)
3. [Transport & Connection](#transport--connection)
4. [Tools Reference](#tools-reference)
   - [url_scanner.scan](#url_scannerscan)
     - [URL Validation](#url-validation)
   - [url_scanner.job_status](#url_scannerjob_status)
   - [url_scanner.cancel_job](#url_scannercancel_job)
   - [url_scanner.queue_stats](#url_scannerqueue_stats)
5. [Classification Results](#classification-results)
6. [HTTP Status Codes](#http-status-codes)
7. [Error Codes](#error-codes)
   - [JSON-RPC Standard Errors](#json-rpc-standard-errors)
   - [Authentication & Rate Limit Errors](#authentication--rate-limit-errors)
   - [URL Validation Errors (Request-Level)](#url-validation-errors-request-level)
   - [Scan Failure Error Codes (Job-Level)](#scan-failure-error-codes-job-level)
8. [Example Workflows](#example-workflows)
9. [Rate Limits & Quotas](#rate-limits--quotas)
10. [Support](#support)

---

## Overview

The URL Scanner MCP Server provides AI-powered malicious URL detection through the Model Context Protocol (MCP). 

### Key Characteristics

| Property             | Value         |
|----------------------|---------------|
| Typical Scan Time    | 20-60 seconds |
| Supported Schemes    | HTTP, HTTPS   |

---

## Authentication

All API requests require authentication via the `X-API-Key` header.

```http
X-API-Key: your-api-key-here
```

### Obtaining API Keys

Contact the service administrator to get API keys.

### Authentication Errors

| Status | Error        | Description                |
|--------|--------------|----------------------------|
| 401    | Unauthorized | Missing or invalid API key |

---

## Transport & Connection

### Streamable HTTP (Production)

**Endpoint**: `https://urlcheck.ai/mcp`

| Method   | Path      | Description                |
|----------|-----------|----------------------------|
| `POST`   | `/mcp`    | JSON-RPC requests          |
| `GET`    | `/mcp`    | SSE stream (notifications) |
| `DELETE` | `/mcp`    | Session termination        |

### Required Headers (MCP 2025-06-18)

| Header                 | Required   | Description                                                           |
|------------------------|------------|-----------------------------------------------------------------------|
| `Content-Type`         | POST       | Must be `application/json`                                            |
| `Accept`               | POST/GET   | POST: `application/json, text/event-stream`; GET: `text/event-stream` |
| `X-API-Key`            | All        | Your API key for authentication                                       |
| `Mcp-Session-Id`       | After init | Session ID returned by `initialize`                                   |
| `MCP-Protocol-Version` | After init | Protocol version (e.g., `2025-06-18`)                                 |

### Session Management

Sessions are managed via the `Mcp-Session-Id` header:

1. Server returns `Mcp-Session-Id` in response to `initialize`
2. Include this header in all later requests
3. Include `MCP-Protocol-Version` header in all subsequent requests
4. Sessions can be terminated via `DELETE /mcp`

### Example: Initialize Session

```bash
curl -X POST https://urlcheck.ai/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2025-06-18",
      "capabilities": {},
      "clientInfo": {"name": "my-client", "version": "1.0"}
    }
  }'
# Response includes Mcp-Session-Id header
```

---

## Tools Reference

### url_scanner.scan

Start an asynchronous URL security scan. Returns immediately with a job ID for polling.

**This is the recommended approach for production use.**

#### Input Schema

```json
{
  "type": "object",
  "properties": {
    "url": {
      "type": "string",
      "description": "The URL to analyze. Must be HTTP or HTTPS. If no scheme provided, https:// is assumed."
    }
  },
  "required": ["url"]
}
```

#### Output Schema (Success)

```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "queued",
  "message": "Scan job queued. Poll url_scanner.job_status with job_id for results.",
  "url": "https://example.com",
  "estimated_wait_secs": 30
}
```

#### Output Schema (Error)

```json
{
  "error": true,
  "error_code": "QUEUE_FULL",
  "message": "Job queue is full. Try again later.",
  "retryable": true,
  "retry_after_secs": 30
}
```

#### Example Request

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "url_scanner.scan",
    "arguments": {
      "url": "https://suspicious-site.com/login"
    }
  }
}
```

#### URL Validation

All URLs submitted to `url_scanner.scan` undergo comprehensive validation before processing. Invalid URLs are rejected immediately with a JSON-RPC error.

##### URL Requirements

| Requirement     | Value           | Description                 |
|-----------------|-----------------|-----------------------------|
| Allowed schemes | `http`, `https` | Other schemes are rejected  |
| Default scheme  | `https://`      | Added if no scheme provided |
| Length limits   | Enforced        | Standard browser limits     |

##### Security Validations

The server performs security checks to protect against malicious input:

| Check                    | Description                                    |
|--------------------------|------------------------------------------------|
| **Scheme validation**    | Only standard web schemes are permitted        |
| **Injection prevention** | URLs containing attack patterns are rejected   |
| **Internal access**      | Private/internal network targets are blocked   |
| **Encoding validation**  | Malformed or suspicious encodings are rejected |

> **Note:** Validation rules are subject to change without notice as new threats emerge.

##### Validation Error Response

URL validation errors return **HTTP 200** with a **JSON-RPC error code `-32602`** (Invalid params).

This follows the MCP/JSON-RPC 2.0 specification:
- The HTTP request itself is valid (headers, session, envelope)
- The error is in the method *parameters* (the URL), not the request structure

**Example: Validation Error**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32602,
    "message": "Invalid params",
    "data": "URL validation failed"
  }
}
```

---

### url_scanner.job_status

Get the status and results of a scan job. Poll this after calling `scan`.

#### Input Schema

```json
{
  "type": "object",
  "properties": {
    "job_id": {
      "type": "string",
      "description": "The job ID returned by scan"
    }
  },
  "required": ["job_id"]
}
```

#### Output Schema

```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "processing",
  "progress": 45.0,
  "progress_message": "Analyzing page content",
  "next_poll_ms": 2000,
  "url": "https://example.com",
  "created_at": 1733849234,
  "started_at": 1733849236,
  "finished_at": null,
  "elapsed_secs": 15,
  "result": null,
  "error": null
}
```

#### Job Status Values

| Status       | Description                | Terminal |
|--------------|----------------------------|----------|
| `queued`     | Waiting in queue           | No       |
| `processing` | Scan in progress           | No       |
| `completed`  | Scan finished successfully | Yes      |
| `failed`     | Scan failed with error     | Yes      |
| `cancelled`  | Scan cancelled by user     | Yes      |

#### Progress Tracking

The `progress` field (0-100) and `progress_message` provide scan progress indication for UI display.

| Phase         | Description                      |
|---------------|----------------------------------|
| `queued`      | Waiting for available worker     |
| `processing`  | Active analysis in progress      |
| `completed`   | Scan finished, results available |

**Polling Recommendation:** Use the `next_poll_ms` field from the response to determine an optimal polling interval. This value adapts based on the current scan phase.

#### Completed Result Structure

When `status` is `completed`, the `result` field contains:

```json
{
  "result": {
    "classification": "Malicious",
    "risk_score": 0.87,
    "confidence": 0.92,
    "analysis_complete": true
  }
}
```

##### Result Fields

| Field               | Type     | Description                                                                     |
|---------------------|----------|---------------------------------------------------------------------------------|
| `classification`    | string   | Risk classification: `Malicious`, `Suspicious`, `Undetected`, `Harmless`        |
| `risk_score`        | number   | Threat probability from 0.0 (safe) to 1.0 (dangerous)                           |
| `confidence`        | number   | Classification confidence from 0.0 to 1.0                                       |
| `analysis_complete` | boolean  | Whether all scanners completed successfully                                     |
| `partial_analysis`  | string[] | Scanner components that failed (only present when `analysis_complete` is false) |

##### Partial Analysis Example

When some scanners fail (e.g., SSL connection timeout), the result indicates which components are missing:

```json
{
  "result": {
    "classification": "Suspicious",
    "risk_score": 0.65,
    "confidence": 0.75,
    "analysis_complete": false,
    "partial_analysis": ["ssl"]
  }
}
```

**Interpretation:** The scan completed but SSL certificate analysis failed. The classification is based on available data only. Consumers should treat partial results with appropriate caution.

#### Failure Response Structure

When `status` is `failed`, the response includes structured error information for programmatic handling:

```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "failed",
  "error": "Domain does not exist",
  "error_code": "DNS_NXDOMAIN",
  "details": {
    "stage": "dns",
    "retryable": false,
    "suggested_action": "stop",
    "retry_after_secs": null
  },
  "stage": "failed",
  "elapsed_secs": 2
}
```

##### Error Response Fields

| Field                      | Type         | Description                                        |
|----------------------------|--------------|----------------------------------------------------|
| `error`                    | string       | Human-readable error message                       |
| `error_code`               | string       | Machine-readable error code (e.g., `DNS_NXDOMAIN`) |
| `details.stage`            | string       | Pipeline stage where failure occurred              |
| `details.retryable`        | boolean      | Whether the operation can be retried               |
| `details.suggested_action` | string       | `stop`, `retry_backoff`, or `treat_as_suspicious`  |
| `details.retry_after_secs` | number\|null | Wait time before retry (null if not retryable)     |
| `suspicious_by_policy`     | boolean      | Present when fail-closed policy triggered          |

##### Suggested Actions

| Action                | Meaning                    | Recommended Client Behavior                     |
|-----------------------|----------------------------|-------------------------------------------------|
| `stop`                | Do not retry               | Move on, log failure                            |
| `retry_backoff`       | Retry with backoff         | Wait `retry_after_secs`, retry (max 3 attempts) |
| `treat_as_suspicious` | Consider failure as signal | Log as potential threat indicator               |

> **Note:** See [Scan Failure Error Codes](#scan-failure-error-codes) for the complete error code reference.

---

### url_scanner.cancel_job

Cancel a queued or running scan job.

#### Input Schema

```json
{
  "type": "object",
  "properties": {
    "job_id": {
      "type": "string",
      "description": "The job ID to cancel"
    }
  },
  "required": ["job_id"]
}
```

#### Output Schema (Success)

```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "cancelled": true,
  "message": "Job cancelled successfully",
  "status": "cancelled"
}
```

#### Output Schema (Error)

```json
{
  "error": true,
  "error_code": "JOB_ALREADY_TERMINAL",
  "message": "Job is already in terminal state: completed",
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "retryable": false
}
```

---

### url_scanner.queue_stats

Get statistics about the job queue. Useful for monitoring capacity before submitting jobs.

#### Input Schema

```json
{
  "type": "object",
  "properties": {},
  "required": []
}
```

#### Output Schema

```json
{
  "queued": 5,
  "processing": 2,
  "completed": 127,
  "failed": 3,
  "cancelled": 2,
  "estimated_wait_secs": 45
}
```

> **Note:** Use `estimated_wait_secs` to inform users of expected wait times. Current counts help determine if the service is under load.

---

## Classification Results

### Risk Classifications

| Classification | Risk Score Range | Description                                  |
|----------------|------------------|----------------------------------------------|
| `Harmless`     | 0.00 - 0.20      | URL explicitly found to be safe              |
| `Undetected`   | 0.20 - 0.40      | No known threats detected                    |
| `Suspicious`   | 0.40 - 0.70      | Unusual behavior, not confirmed as malicious |
| `Malicious`    | 0.70 - 1.00      | URL flagged as dangerous                     |

### Risk Score

A float between 0.0 and 1.0 indicating threat probability:
- **0.0**: Safe
- **0.5**: Uncertain
- **1.0**: Dangerous

---

## HTTP Status Codes

The MCP server follows the MCP 2025-06-18 specification for HTTP status codes.

### Success Codes (2xx)

| Code | Status      | When Returned                                  |
|------|-------------|------------------------------------------------|
| 200  | OK          | Successful request with JSON or SSE response   |
| 202  | Accepted    | Notification received (no response body)       |
| 204  | No Content  | DELETE /mcp session termination successful     |

### Client Error Codes (4xx)

| Code | Status                 | When Returned                                           |
|------|------------------------|---------------------------------------------------------|
| 400  | Bad Request            | Invalid JSON, missing headers, invalid protocol version |
| 401  | Unauthorized           | Missing or invalid `X-API-Key` header                   |
| 404  | Not Found              | Unknown/expired session ID, wrong endpoint              |
| 415  | Unsupported Media Type | Missing or wrong `Content-Type` header                  |
| 429  | Too Many Requests      | Rate limit exceeded                                     |

### Server Error Codes (5xx)

| Code | Status                | When Returned                 |
|------|-----------------------|-------------------------------|
| 500  | Internal Server Error | Unhandled server exception    |
| 503  | Service Unavailable   | Queue full, server overloaded |

### Example Error Responses

**401 Unauthorized** (missing API key):

Authentication errors return plain HTTP responses (not JSON-RPC):
```
HTTP/1.1 401 Unauthorized
Content-Type: text/plain

missing api key
```

**429 Rate Limited** (brute-force protection):

Rate limiting from failed auth attempts returns plain HTTP:
```
HTTP/1.1 429 Too Many Requests
Content-Type: text/plain

too many failed attempts, retry after 60 seconds
```

**429 Rate Limited** (API rate limit after successful auth):

Rate limiting after authentication returns JSON-RPC with `Retry-After` header:
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": 1022,
    "message": "Rate limit exceeded. Retry after 5000 ms"
  },
  "id": 1
}
```

**400 Bad Request** (missing Accept header):
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32600,
    "message": "Accept header must include application/json or text/event-stream"
  },
  "id": null
}
```

---

## Error Codes

### Tool-Level Errors

Tool-level errors are returned as **successful JSON-RPC responses** with the error embedded in the tool result content. The response has `isError: true` and the content contains a JSON object with `error: true` and an `error_code` field.

**Example response structure:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [{
      "type": "text",
      "text": "{\"error\": true, \"error_code\": \"QUEUE_FULL\", \"message\": \"...\", \"retryable\": true, \"retry_after_secs\": 30}"
    }],
    "isError": true
  }
}
```

| Code                   | Description                            | Retryable |
|------------------------|----------------------------------------|-----------|
| `QUEUE_FULL`           | Job queue at capacity                  | Yes (30s) |
| `KEY_QUOTA_EXCEEDED`   | API key has too many active jobs       | Yes (30s) |
| `JOB_NOT_FOUND`        | Job ID does not exist                  | No        |
| `JOB_ALREADY_TERMINAL` | Job already completed/failed/cancelled | No        |

> **Note:** These are NOT JSON-RPC error codes. Check for `result.isError == true` and parse the `error_code` from the text content.

### URL Validation Errors

URL validation failures return JSON-RPC error code **`-32602`** (Invalid params) with information in the `data` field.

| Category           | Description                                      |
|--------------------|--------------------------------------------------|
| Format errors      | Empty, malformed, or excessively long URLs       |
| Scheme errors      | Unsupported or potentially dangerous URL schemes |
| Security errors    | URLs containing patterns that may indicate abuse |

> **Note:** Error messages provide general feedback without disclosing specific validation rules.

### JSON-RPC Standard Errors

| Code   | Message          | HTTP Status | Description                                          |
|--------|------------------|-------------|------------------------------------------------------|
| -32700 | Parse error      | 400         | Invalid JSON syntax                                  |
| -32600 | Invalid request  | 400         | Malformed JSON-RPC envelope                          |
| -32601 | Method not found | 200         | Unknown method name                                  |
| -32602 | Invalid params   | 200         | Invalid method parameters (including URL validation) |
| -32603 | Internal error   | 500         | Unexpected server error                              |

### Authentication & Rate Limit Errors

**Note:** Authentication errors return plain HTTP responses (not JSON-RPC envelopes).

| HTTP Status | Response Type | Body                                                                 | Description             |
|-------------|---------------|----------------------------------------------------------------------|-------------------------|
| 401         | Plain text    | `missing api key`                                                    | No API key provided     |
| 401         | Plain text    | `invalid api key`                                                    | API key not recognized  |
| 401         | Plain text    | `api key expired`                                                    | API key has expired     |
| 403         | Plain text    | `ip blocked`                                                         | IP address is blocked   |
| 403         | Plain text    | `origin not allowed`                                                 | CORS origin rejected    |
| 429         | Plain text    | `too many failed attempts, retry after X seconds`                    | Brute-force protection  |
| 429         | JSON-RPC      | `{"code": 1022, "message": "Rate limit exceeded. Retry after X ms"}` | API rate limit exceeded |

### URL Validation Errors (Request-Level)

URL validation errors occur **before** a job is created. These return a standard JSON-RPC `-32602 Invalid params` error, not a job failure.

| Validation Error    | Example Message                                         |
|---------------------|---------------------------------------------------------|
| Empty/too short URL | `URL too short: 3 characters (minimum 5)`               |
| URL too long        | `URL too long: 2500 characters (maximum 2048)`          |
| Invalid scheme      | `Invalid URL scheme 'ftp'. Only HTTP and HTTPS allowed` |
| Missing host        | `URL must have a valid host`                            |
| Dangerous scheme    | `Dangerous URL scheme 'javascript' is not allowed`      |
| Injection attempt   | `Potential XSS injection detected: <script>`            |
| SSRF attempt        | `Obfuscated IP address detected: decimal encoding`      |

**Note:** These errors prevent job creation—no `job_id` is returned and no `job_status` entry exists.

---

### Scan Failure Error Codes (Job-Level)

When a scan job fails during execution, the `job_status` response contains an `error_code` field with a machine-readable code. These are **job-level failures** that occur after the job has been created and started processing.

#### Network & DNS Errors

| Error Code              | Message                 | Retryable | Action        |
|-------------------------|-------------------------|-----------|---------------|
| `DNS_NXDOMAIN`          | Domain does not exist   | No        | stop          |
| `DNS_RESOLUTION_FAILED` | DNS lookup failed       | Yes (30s) | retry_backoff |
| `CONNECTION_REFUSED`    | Connection refused      | No        | stop          |
| `CONNECTION_TIMEOUT`    | TCP handshake timed out | Yes (60s) | retry_backoff |
| `NETWORK_UNREACHABLE`   | No route to host        | Yes (60s) | retry_backoff |

#### TLS/SSL Errors

| Error Code               | Message                     | Retryable | Action              |
|--------------------------|-----------------------------|-----------|---------------------|
| `TLS_HANDSHAKE_FAILED`   | SSL/TLS handshake failed    | No        | stop                |
| `TLS_CERT_EXPIRED`       | Certificate expired         | No        | treat_as_suspicious |
| `TLS_CERT_INVALID`       | Certificate invalid         | No        | treat_as_suspicious |
| `TLS_CERT_NAME_MISMATCH` | Certificate domain mismatch | No        | treat_as_suspicious |

#### HTTP Protocol Errors

| Error Code           | Message                    | Retryable  | Action              |
|----------------------|----------------------------|------------|---------------------|
| `HTTP_ACCESS_DENIED` | Access denied (401/403)    | No         | treat_as_suspicious |
| `HTTP_NOT_FOUND`     | Page not found (404)       | No         | stop                |
| `TARGET_RATE_LIMIT`  | Target rate limiting (429) | Yes (120s) | retry_backoff       |
| `REDIRECT_LOOP`      | Too many redirects         | No         | treat_as_suspicious |
| `EMPTY_RESPONSE`     | Empty response             | Yes (30s)  | retry_backoff       |

#### Browser & Rendering Errors

| Error Code                  | Message                       | Retryable | Action              |
|-----------------------------|-------------------------------|-----------|---------------------|
| `NAVIGATION_TIMEOUT`        | Page load timeout             | Yes (60s) | retry_backoff       |
| `ANTI_BOT_BLOCK`            | Blocked by anti-automation    | No        | treat_as_suspicious |
| `UNSUPPORTED_CONTENT_TYPE`  | Not a webpage (PDF, etc.)     | No        | stop                |
| `RENDERER_CRASHED`          | Browser tab crashed           | No        | treat_as_suspicious |
| `BROWSER_UNRESPONSIVE`      | Browser unresponsive          | Yes (60s) | treat_as_suspicious |
| `DOM_DETACHED`              | Page closed via script        | No        | treat_as_suspicious |
| `BROWSER_INIT_FAILED`       | Browser initialization failed | Yes (30s) | retry_backoff       |
| `BROWSER_UNKNOWN_NET_ERROR` | Unknown network error         | Yes (30s) | retry_backoff       |
| `TRANSPORT_TIMEOUT`         | Browser communication timeout | Yes (60s) | retry_backoff       |

#### System & Queue Errors

| Error Code           | Message                | Retryable | Action        |
|----------------------|------------------------|-----------|---------------|
| `JOB_TIMEOUT`        | Analysis time exceeded | No        | stop          |
| `QUEUE_FULL`         | Queue at capacity      | Yes (30s) | retry_backoff |
| `JOB_EXPIRED`        | Job expired in queue   | Yes (10s) | retry_backoff |
| `KEY_QUOTA_EXCEEDED` | API key limit reached  | Yes (60s) | retry_backoff |
| `JOB_CANCELLED`      | Cancelled by user      | No        | stop          |
| `INTERNAL_ERROR`     | Internal exception     | Yes (30s) | retry_backoff |

#### Error Stages

These stages indicate where in the pipeline a failure occurred (used in `details.stage`):

| Stage        | Description                    |
|--------------|--------------------------------|
| `preflight`  | URL validation, SSRF checks    |
| `dns`        | Domain name resolution         |
| `network`    | TCP connection                 |
| `tls`        | SSL/TLS handshake              |
| `http`       | HTTP protocol exchange         |
| `navigation` | Browser page navigation        |
| `render`     | Page rendering and DOM         |
| `system`     | Job orchestration/system level |

#### Internal Error Code Ranges (Developer Reference)

**String vs Numeric Codes:**
- **Tool results** (`url_scanner.scan`, `url_scanner.job_status`) return **string** `error_code` values (e.g., `QUEUE_FULL`, `DNS_NXDOMAIN`)
- **JSON-RPC `error.code`** uses **numeric** codes for protocol errors, authentication, and rate limiting

**Numeric Code Ranges:**

| Range            | Enum                       | Context                                                               |
|------------------|----------------------------|-----------------------------------------------------------------------|
| -32700 to -32600 | JSON-RPC standard          | Protocol errors (parse, invalid request, method not found, etc.)      |
| 1000–1099        | `McpServerErrorCode`       | Server/auth errors in JSON-RPC `error.code` (e.g., 1022 = rate limit) |
| 2000–2099        | `ValidationErrorCode`      | Input validation (mapped to `-32602` in JSON-RPC responses)           |
| 3000–3099        | `UrlInspectorErrorCode`    | Scan failures (returned as strings in `job_status.error_code`)        |
| 3100–3119        | `UrlInspectorErrorCode`    | Job/system errors (timeout, queue full, cancelled)                    |
| 4000–4099        | `HybridPredictorErrorCode` | ML prediction errors (internal)                                       |

> **Client Implementation:** Parse `error_code` as a **string** from tool results and `job_status` responses. Only JSON-RPC `error.code` fields contain numeric codes.
>
> **Note:** URL validation errors (invalid URL, unsupported scheme, SSRF attempts) are caught at request time and return `-32602 Invalid params`—they do not create jobs or appear in `job_status`.

---

## Example Workflows

### Basic Scan Workflow

```
1. Call url_scanner.scan(url: "https://suspicious.com")
   → Returns: { job_id: "abc123", status: "queued" }

2. Poll url_scanner.job_status(job_id: "abc123")
   → Returns: { status: "processing", progress: 45, progress_message: "Analyzing page..." }

3. Continue polling until status is terminal...
   → Returns: { status: "completed", result: { classification: "Malicious", ... } }
```

### Checking Capacity Before Scanning

```
1. Call url_scanner.queue_stats()
   → Returns: { queued: 15, processing: 2, estimated_wait_secs: 45 }

2. Use estimated_wait_secs to inform user of expected wait time
   If queue appears heavily loaded, consider notifying the user
```

### Handling Rate Limits

```python
async def scan_with_retry(url, max_retries=3):
    for attempt in range(max_retries):
        result = await scan(url)

        if result.get("error_code") == "QUEUE_FULL":
            wait_time = result.get("retry_after_secs", 30)
            await asyncio.sleep(wait_time)
            continue

        return result

    raise Exception("Queue full after max retries")
```

---

## Rate Limits & Quotas

The API enforces rate limits to ensure fair usage and service availability. Limits may be adjusted without notice based on operational requirements.

### Per-API-Key Limits

| Limit               | Description                        |
|---------------------|------------------------------------|
| Requests per minute | Enforced per API key               |
| Concurrent jobs     | Limited active jobs per API key    |

### Global Limits

| Limit           | Description                    |
|-----------------|--------------------------------|
| Queue capacity  | Total pending jobs is limited  |
| Job timeout     | Long-running scans are stopped |

> **Note:** Your API key tier determines specific limit values. Contact support for details on available tiers.

### Rate Limit Responses

There are two types of rate limiting with different response formats:

**1. Brute-Force Protection (Plain HTTP 429)**

When too many authentication failures occur from the same IP, the server returns a plain HTTP response:

```
HTTP/1.1 429 Too Many Requests
Content-Type: text/plain
Retry-After: 60

too many failed attempts, retry after 60 seconds
```

**2. API Rate Limit (JSON-RPC 1022)**

When the API quota is exceeded after successful authentication, the server returns a JSON-RPC error:

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": 1022,
    "message": "Rate limit exceeded. Retry after 1000 ms"
  },
  "id": 1
}
```

The response includes a `Retry-After` header with the retry delay in seconds.

---

## Support

For API keys, support, or questions:
- **Publisher**: [CybrLab AI](https://cybrlab.ai)
- **Service**: [urlcheck.ai](https://urlcheck.ai)
- **Email**: contact@cybrlab.ai
