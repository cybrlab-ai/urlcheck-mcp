# Authentication Guide

> How to get and use API keys for the URL Security Scanner MCP Server

**Publisher:** [CybrLab AI](https://cybrlab.ai) | **Service:** [urlcheck.ai](https://urlcheck.ai)

---

## API Key Authentication

All requests to the URL Security Scanner require authentication via the `X-API-Key` header.

### Request Format

```http
POST /mcp HTTP/1.1
Host: urlcheck.ai
Content-Type: application/json
Accept: application/json, text/event-stream
X-API-Key: your-api-key-here
MCP-Protocol-Version: 2025-06-18
Mcp-Session-Id: your-session-id

{...}
```

### curl Example (Initialize Session)

```bash
# First request: initialize (no session ID needed yet)
curl -X POST https://urlcheck.ai/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "X-API-Key: your-api-key-here" \
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
# Response includes Mcp-Session-Id header - save it!
```

### curl Example (Subsequent Requests)

```bash
# All requests after initialize require session ID and protocol version
curl -X POST https://urlcheck.ai/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "X-API-Key: your-api-key-here" \
  -H "MCP-Protocol-Version: 2025-06-18" \
  -H "Mcp-Session-Id: your-session-id" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
      "name": "url_scanner.queue_stats",
      "arguments": {}
    }
  }'
```

---

## Obtaining API Keys

### For Production Use

Contact CybrLab AI to get production API keys:

- **Email**: contact@cybrlab.ai
- **Website**: [cybrlab.ai](https://cybrlab.ai)

### For Testing/Evaluation

Contact us for evaluation keys with limited quotas for testing purposes.

---

## Authentication Errors

Authentication failures return **plain HTTP responses** (not JSON-RPC):

| HTTP Status | Response Type | Body                                              | Description                      |
|-------------|---------------|---------------------------------------------------|----------------------------------|
| 401         | Plain text    | `missing api key`                                 | No `X-API-Key` header provided   |
| 401         | Plain text    | `invalid api key`                                 | API key not recognized           |
| 429         | Plain text    | `too many failed attempts, retry after X seconds` | Brute-force protection triggered |

### Error Response Examples

**Missing API Key (401):**
```
HTTP/1.1 401 Unauthorized
Content-Type: text/plain

missing api key
```

**Invalid API Key (401):**
```
HTTP/1.1 401 Unauthorized
Content-Type: text/plain

invalid api key
```

**Brute-Force Protection (429):**
```
HTTP/1.1 429 Too Many Requests
Content-Type: text/plain
Retry-After: 60

too many failed attempts, retry after 60 seconds
```

> **Note:** API rate limits (quota exceeded after successful auth) return JSON-RPC errors with code `1022` and include a `Retry-After` header. See the [API Reference](./API.md#rate-limits--quotas) for details.

---

## Best Practices

1. **Never expose API keys** in client-side code or public repositories
2. **Implement retry logic** for rate limit errors (429)
3. **Monitor queue capacity** via `url_scanner.queue_stats` tool before submitting jobs

### Environment Variable Example

```bash
export URLCHECK_API_KEY="your-api-key-here"
```

```python
import os

api_key = os.environ.get("URLCHECK_API_KEY")
headers = {"X-API-Key": api_key}
```

---

## Support

For API key issues or quota increases:

- **Publisher**: [CybrLab AI](https://cybrlab.ai)
- **Email**: contact@cybrlab.ai
