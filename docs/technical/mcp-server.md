# MCP Server — FEL AI Studio

## Responsibility

`server_stdio.py` is the system’s MCP server. It exposes three FEL tools (`fel_validate`, `fel_render`, `fel_batch`) as functions callable by the AI agent through the MCP protocol over JSON-RPC 2.0 via stdio.

It is a separate process from the FastAPI backend. It is launched as a subprocess by `McpStdioClient` and communicates exclusively through stdin/stdout. It has no knowledge of sessions, JWTs, or users: it only receives arguments, executes deterministic operations, and returns results.

---

## Location in the architecture

```bash id="c6mp06"
ChatEngine
    └── McpStdioClient
            │  JSON-RPC 2.0 over stdio
            ▼
        server_stdio.py     ← this module
            ├── fel_validate  → validateFel()
            ├── fel_render    → renderFel()
            └── fel_batch     → batchFel()
                    │
                    ▼
                fel_pdf.py
                    ├── readFelXml()
                    └── generatePdf()
```

---

## Protocol

### Transport

JSON-RPC 2.0 over stdio. One message per line. No SDK.

```bash id="5t6r73"
stdin  → client requests (ChatEngine)
stdout → server responses
stderr → diagnostic logs (not consumed by client)
```

### Request format

```json id="95v43u"
{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tools/call",
    "params": {
        "name": "fel_render",
        "arguments": {
            "xml_path": "/tmp/fel/<session_uuid>/<file_uuid>.xml",
            "out_path": "/tmp/fel/<session_uuid>/out_<uuid>.pdf"
        }
    }
}
```

### Successful response format

```json id="id81xq"
{
    "jsonrpc": "2.0",
    "id": 3,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "{\"ok\": true, \"pdf_path\": \"/tmp/fel/...\"}"
            }
        ]
    }
}
```

The result of each tool is sent as serialized JSON inside the `text` field. The client (`parseTextBlock`) deserializes it.

### Error response format

```json id="34ajmd"
{
    "jsonrpc": "2.0",
    "id": 3,
    "error": {
        "code": -32000,
        "message": "out_path must be a non-empty string"
    }
}
```

### Notifications

Messages without `id` (e.g. `notifications/initialized`) are silently ignored. The server does not respond to notifications.

---

## Supported methods

| Method       | Description                             |
| ------------ | --------------------------------------- |
| `initialize` | Initial handshake, returns capabilities |
| `tools/list` | Tool catalog with JSON Schemas          |
| `tools/call` | Tool invocation with arguments          |

Any other method returns error `-32601 Method not found`.

---

## `initialize`

### Request

```json id="m6gacn"
{ "jsonrpc": "2.0", "id": 1, "method": "initialize", "params": {} }
```

### Response

```json id="9yb9ha"
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "protocolVersion": "2025-06-18",
        "serverInfo": { "name": "fel-stdio", "version": "0.1.0" },
        "capabilities": { "tools": { "listChanged": true } }
    }
}
```

---

## `tools/list`

### Response — full catalog

```json id="t6crk0"
{
    "tools": [
        {
            "name": "fel_validate",
            "description": "Validate FEL XML totals (subtotal, VAT 12%, total) and required fields.",
            "inputSchema": {
                "type": "object",
                "properties": {
                    "xml_path": { "type": "string" }
                },
                "required": ["xml_path"]
            }
        },
        {
            "name": "fel_render",
            "description": "Render branded PDF from FEL XML using session visual config.",
            "inputSchema": {
                "type": "object",
                "properties": {
                    "xml_path":  { "type": "string" },
                    "out_path":  { "type": "string" },
                    "logo_path": { "type": ["string", "null"] },
                    "theme":     { "type": ["string", "null"] },
                    "colors": {
                        "type": "object",
                        "properties": {
                            "primary":        { "type": "string" },
                            "primary_text":   { "type": "string" },
                            "secondary":      { "type": "string" },
                            "secondary_text": { "type": "string" },
                            "text_main":      { "type": "string" },
                            "text_soft":      { "type": "string" },
                            "background":     { "type": "string" }
                        }
                    }
                },
                "required": ["xml_path", "out_path"]
            }
        },
        {
            "name": "fel_batch",
            "description": "Render all FEL XMLs in a directory. Outputs one PDF per file and manifest.json.",
            "inputSchema": {
                "type": "object",
                "properties": {
                    "dir_xml": { "type": "string" },
                    "out_dir": { "type": "string" }
                },
                "required": ["dir_xml", "out_dir"]
            }
        }
    ]
}
```

---

## Tools

### `fel_validate`

Validates internal consistency of a FEL XML:

* Computes expected VAT: `subtotal × 0.12` (rounded to 2 decimals)
* Computes expected total: `subtotal + VAT`
* Compares with tolerance of `±0.01`
* Verifies presence of required fields

**Arguments:**

| Field      | Type   | Required |
| ---------- | ------ | -------- |
| `xml_path` | string | Yes      |

**Successful response:**

```json id="t3zt28"
{
    "ok": true,
    "issues": [],
    "totals": {
        "subtotal": "8010.59",
        "iva": "961.27",
        "total": "8971.86"
    }
}
```

**Response with issues:**

```json id="4c5w66"
{
    "ok": false,
    "issues": [
        "VAT mismatch: expected 961.27 got 960.00",
        "Missing field: numero_autorizacion"
    ],
    "totals": {
        "subtotal": "8010.59",
        "iva": "960.00",
        "total": "8970.59"
    }
}
```

**Note:** `ok=false` does not block rendering. Claude evaluates issues and decides whether to proceed (Variant 1, ADR 0002).

---

### `fel_render`

Generates a customized PDF from a FEL XML using `PdfConfig`.

**Arguments:**

| Field       | Type          | Required | Default                 |
| ----------- | ------------- | -------- | ----------------------- |
| `xml_path`  | string        | Yes      | —                       |
| `out_path`  | string        | Yes      | —                       |
| `logo_path` | string \| null | No       | `LOGO_PATH` from config |
| `colors`    | object        | No       | `PdfConfig` defaults    |
| `theme`     | string \| null | No       | reserved for v2         |

**`PdfConfig` construction on server:**

In web mode, colors come from `arguments.colors` built from the JWT by ChatEngine.

In Docker mode, colors follow ADR 0006 precedence:

```bash id="ntih9r"
arguments.colors > fel_config.json > env vars > defaults
```

**Successful response:**

```json id="znt6yx"
{
    "ok": true,
    "pdf_path": "/tmp/fel/<session_uuid>/out_<uuid>.pdf"
}
```

**Pre-render validations:**

* `xml_path` must be a non-empty string
* `out_path` must be a non-empty string
* Both paths are validated against the sandbox before execution

---

### `fel_batch`

Processes all `*.xml` files in a directory, generates one PDF per file, and writes `manifest.json` mapping `xml → pdf`.

**Arguments:**

| Field     | Type   | Required |
| --------- | ------ | -------- |
| `dir_xml` | string | Yes      |
| `out_dir` | string | Yes      |

**Successful response:**

```json id="wgsq49"
{
    "ok": true,
    "count": 5,
    "out_dir": "/tmp/fel/<session_uuid>/batch/",
    "manifest_path": "/tmp/fel/<session_uuid>/batch/manifest.json"
}
```

**Note:** `fel_batch` uses a single `PdfConfig` for all files. No per-XML customization.

---

## Authentication in Docker Hub mode

In Docker mode, the server verifies `MCP_TOKEN` before processing any request:

```bash id="e4ejyr"
Incoming request
    └── params.mcp_token present and valid?
          ├── Yes → process normally
          └── No → error -32001 "Unauthorized"
```

In web mode (backend subprocess), no token verification is required: the server runs as an isolated child process, not exposed to the network.

---

## Main loop `main()`

```bash id="y1dz20"
For each line from stdin:
    1. json.loads(line) → req
    2. Extract method and id
    3. If no id → notification → ignore silently
    4. Dispatch by method:
       - "initialize"  → sendResponse(id, getCapabilities())
       - "tools/list"  → sendResponse(id, listTools())
       - "tools/call"  → callTool(name, arguments)
                          → wrap in content envelope
                          → sendResponse(id, wrapper)
       - other method  → error -32601
    5. Global exception → error -32000 with str(e)
                          only if id is valid
```

---

## `McpStdioClient` — the client

Client used by `ChatEngine` to communicate with this server.

### Design

* Launches server via `subprocess.Popen`
* A daemon thread continuously reads stdout and queues lines
* `rpc()` writes request to stdin and waits for response from queue
* Correlates by `id`: discards unrelated or non-JSON lines
* One in-flight request at a time in this implementation

### Exposed methods

| Method                 | Description                                  |
| ---------------------- | -------------------------------------------- |
| `start()`              | Launch subprocess + thread + send initialize |
| `stop()`               | Terminate subprocess                         |
| `listTools()`          | JSON-RPC `tools/list`                        |
| `callTool(name, args)` | JSON-RPC `tools/call`                        |

### Timeout

`startupTimeoutSec = 8.0` by default. If the server does not respond in time, raises `TimeoutError`.

---

## `rpc_logger` — traceability

All JSON-RPC messages (sent and received) are logged as JSONL files under `LOG_RPC/`:

```bash id="smllz6"
logs/rpc/mcp_rpc_<YYYYMMDD_HHMMSS>.jsonl
```

Each line:

```json id="fghqox"
{
    "time": "2025-03-17T10:23:45.123456",
    "direction": "send" | "recv",
    "data": { ...full payload... }
}
```

Always active, independent of `ROUTER_DEBUG`. Primary tool for debugging protocol issues.

---

## Testing the server in isolation

The server can be tested directly from terminal without the backend:

```bash id="o6dciv"
# Initialize + list tools
printf '%s\n' \
'{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}' \
'{"jsonrpc":"2.0","id":2,"method":"tools/list"}' \
| python server_stdio.py

# Validate XML
printf '%s\n' \
'{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}' \
'{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"fel_validate","arguments":{"xml_path":"data/xml/factura.xml"}}}' \
| python server_stdio.py

# Generate PDF
printf '%s\n' \
'{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}' \
'{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"fel_render","arguments":{"xml_path":"data/xml/factura.xml","out_path":"data/out/factura.pdf"}}}' \
| python server_stdio.py
```

This isolated testing capability is a key advantage of the MCP architecture (ADR 0002).

---

## Differences from previous code

| Aspect            | Previous code                   | This version                |
| ----------------- | ------------------------------- | --------------------------- |
| `out_path`        | Optional (default in config.py) | Explicitly required         |
| Colors            | Hardcoded in config.py          | Passed as `colors` argument |
| `PdfConfig`       | Did not exist                   | Built from arguments        |
| Authentication    | No token                        | `MCP_TOKEN` in Docker mode  |
| `protocolVersion` | `2024-11-05`                    | `2025-06-18`                |
| Notifications     | Not handled                     | Silently ignored            |

---

## Related files

* `adr/0002-mcp-as-orchestration-layer.md` → MCP architecture decision
* `adr/0006-docker-hub-mcp-distribution.md` → authentication and config in Docker
* `technical/engine.md` → ChatEngine and McpStdioClient
* `technical/pdf-generation.md` → generatePdf and PdfConfig
* `architecture/flow-diagram.md` → MCP protocol diagrams
