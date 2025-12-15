# URL Security Scanner MCP Server

> **Malicious URL Detection for Safe Agentic Browsing**
>
> High-accuracy MCP-native URL scanner for secure agentic browsing, powered by ML classification and browser automation.

**Publisher:** [CybrLab AI](https://cybrlab.ai) | **Service:** [urlcheck.ai](https://urlcheck.ai)

---

## Overview

The URL Security Scanner is an MCP (Model Context Protocol) server that provides AI agents with the ability to analyze URLs for malicious content and security threats before navigation.

### Key Features

- **ML Classifier**: Proprietary ML model trained to detect known malicious patterns and generalize to novel(Zero-Day) threats through behavioral/feature-based analysis
- **Browser Automation**: Full page rendering and dynamic content/network analysis via headless browser
- **Brand Impersonation Detection**: Identifies fraudulent pages impersonating brands
- **Heuristics Engine**: Rule-based detection using indicators of compromise and threat signatures
- **GenAI Analysis**: Optional LLM-based reasoning for edge cases requiring contextual analysis

### Use Cases

- Pre-flight URL validation for AI agents
- Automated URL security scanning in workflows
- Malicious link detection in emails/messages

---

## Quick Start

### 1. Configure Your MCP Client

Add to your MCP client configuration:

```json
{
  "mcpServers": {
    "url-scanner": {
      "transport": "streamable-http",
      "url": "https://urlcheck.ai/mcp",
      "headers": {
        "X-API-Key": "YOUR_API_KEY"
      }
    }
  }
}
```

### 2. Initialize Session

```bash
# First, initialize a session (required before any other requests)
curl -X POST https://urlcheck.ai/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "X-API-Key: YOUR_API_KEY" \
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
# Response includes Mcp-Session-Id header - save it for subsequent requests
```

### 3. Start a Scan

```bash
# Use the Mcp-Session-Id from the initialize response
curl -X POST https://urlcheck.ai/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "MCP-Protocol-Version: 2025-06-18" \
  -H "Mcp-Session-Id: YOUR_SESSION_ID" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
      "name": "url_scanner.scan",
      "arguments": {
        "url": "https://example.com"
      }
    }
  }'
```

Response:
```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "queued",
  "estimated_wait_secs": 30
}
```

### 4. Poll for Results

```bash
curl -X POST https://urlcheck.ai/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "MCP-Protocol-Version: 2025-06-18" \
  -H "Mcp-Session-Id: YOUR_SESSION_ID" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tools/call",
    "params": {
      "name": "url_scanner.job_status",
      "arguments": {
        "job_id": "550e8400-e29b-41d4-a716-446655440000"
      }
    }
  }'
```

---

## Available Tools

| Tool                      | Description                   |
|---------------------------|-------------------------------|
| `url_scanner.scan`        | Start async URL security scan |
| `url_scanner.job_status`  | Poll for scan results         |
| `url_scanner.cancel_job`  | Cancel a running scan         |
| `url_scanner.queue_stats` | Get queue statistics          |

See [Full API Documentation](docs/API.md) for detailed schemas and examples.

---

## Authentication

All requests require an API key via the `X-API-Key` header.

See [Authentication Guide](docs/AUTHENTICATION.md) for details on getting API keys.

---

## Technical Specifications

| Property             | Value                     |
|----------------------|---------------------------|
| Protocol             | MCP 2025-06-18            |
| Transport            | Streamable HTTP           |
| Endpoint             | `https://urlcheck.ai/mcp` |
| Typical Scan Time    | 20-60 seconds             |
| Supported Schemes    | HTTP, HTTPS               |
| Max URL Length       | 2048 characters           |

### Input Validation

URLs are validated before processing with comprehensive security checks including scheme validation, injection prevention, and internal network access controls.

Invalid URLs return JSON-RPC error code `-32602` (Invalid params). See [API Documentation](docs/API.md#url-validation) for details.

---

## Support

- **Publisher**: [CybrLab AI](https://cybrlab.ai)
- **Service**: [urlcheck.ai](https://urlcheck.ai)
- **Email**: contact@cybrlab.ai

---

## License

Apache License 2.0 - See [LICENSE](LICENSE) for details.

Copyright 2025 CybrLab AI
