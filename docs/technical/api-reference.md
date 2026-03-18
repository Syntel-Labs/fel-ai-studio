# API Reference — FEL AI Studio

## Base URL

```bash
Development:   http://localhost:8000
Production:    https://api.felai.domain.com
```

---

## Global headers

All endpoints (except `/health`) require these headers:

| Header            | Description                  | Required              |
| ----------------- | ---------------------------- | --------------------- |
| `X-Preview-Token` | System access token          | Yes (production)      |
| `Authorization`   | `Bearer <jwt>` session token | Yes (except /session) |
| `X-API-Key`       | User’s Anthropic API Key     | Only on /chat         |

In `ENV=development`, `X-Preview-Token` may be omitted.

---

## Endpoints

---

### `GET /health`

Server health check.

**Auth required:** none

**Response `200`:**

```json
{
    "status": "ok",
    "env": "development" | "production",
    "mcp": "running" | "stopped"
}
```

---

### `POST /session`

Creates a new session. Generates the signed JWT and the session’s temporary directory in `/tmp/fel/<session_uuid>/`.

**Auth required:** `X-Preview-Token`, `X-API-Key`

**Request body:**

```json
{
    "colors": {
        "primary":        "#fe5101",
        "primary_text":   "#ffffff",
        "secondary":      "#343534",
        "secondary_text": "#ffffff",
        "text_main":      "#000000",
        "text_soft":      "#8c8d8e",
        "background":     "#ffffff"
    },
    "logo_ref":  "logo_<uuid>.jpg",
    "pdf_name":  "factura_{serie}_{dte}.pdf",
    "footer": {
        "website": "https://miempresa.com",
        "phone":   "(502) 2254-9885",
        "email":   "info@miempresa.com"
    },
    "layout": {
        "top_bar_height": 20,
        "qr_size":        150
    }
}
```

All fields are optional except `colors`. If omitted, system defaults are used.

**Response `200`:**

```json
{
    "token": "eyJhbGc...",
    "expires_at": "2025-03-17T11:23:45Z",
    "session_uuid": "a3f9b1c2-..."
}
```

**Response `401`:**

```json
{ "detail": "Missing or invalid X-API-Key" }
```

**Response `422`:**

```json
{
    "detail": [
        {
            "loc": ["body", "colors", "primary"],
            "msg": "Invalid hex color format",
            "type": "value_error"
        }
    ]
}
```

**Notes:**

* `X-API-Key` is validated for format only; it is not sent to Anthropic
* `X-API-Key` is never included in the returned JWT
* `session_uuid` is also included in the JWT payload

---

### `POST /upload`

Uploads the invoice XML to the active session’s temporary directory.

The original filename is discarded and replaced with an internal UUID.

**Auth required:** `X-Preview-Token`, `Authorization: Bearer <jwt>`

**Request:** `multipart/form-data`

| Field  | Type   | Description             |
| ------ | ------ | ----------------------- |
| `file` | `File` | FEL invoice `.xml` file |

**Validations:**

* Extension must be `.xml`
* Maximum size: `MAX_XML_SIZE_KB` (default 500KB)
* Minimal XML structure validated before acceptance

**Response `200`:**

```json
{
    "xml_ref": "7d2e4f1a",
    "xml_path": "/tmp/fel/<session_uuid>/<xml_ref>.xml",
    "size_kb": 12.4
}
```

**Response `400`:**

```json
{ "detail": "Invalid file type. Only .xml files are accepted." }
```

**Response `413`:**

```json
{ "detail": "File exceeds maximum size of 500KB." }
```

**Response `422`:**

```json
{ "detail": "File does not appear to be a valid FEL XML structure." }
```

**Notes:**

* `xml_ref` is used in `/chat` to reference the uploaded file
* XML is not re-sent in each chat message
* The file is deleted when the JWT expires

---

### `POST /chat`

Sends a message to the AI agent. The agent may automatically invoke MCP tools (`fel_validate`, `fel_render`, `fel_batch`).

**Auth required:** `X-Preview-Token`, `Authorization: Bearer <jwt>`, `X-API-Key`

**Request body:**

```json
{
    "message": "Generate the PDF of the invoice I uploaded",
    "xml_ref": "7d2e4f1a",
    "history": [
        { "role": "user",      "content": "Validate my invoice" },
        { "role": "assistant", "content": "The invoice is valid. Subtotal: Q8,010.59..." }
    ]
}
```

| Field     | Type   | Required | Description                           |
| --------- | ------ | -------- | ------------------------------------- |
| `message` | string | Yes      | Current user message                  |
| `xml_ref` | string | No       | Reference to XML uploaded via /upload |
| `history` | array  | No       | Previous messages (text only)         |

**Response `200`:**

```json
{
    "response": "Your invoice has been generated successfully. The PDF includes your logo and configured colors. You can download it using the button below.",
    "pdf_ref": "out_9f3c1b2a",
    "tool_calls": [
        {
            "tool": "fel_validate",
            "args": { "xml_path": "/tmp/fel/.../.xml" },
            "result": { "ok": true, "issues": [], "totals": {...} }
        },
        {
            "tool": "fel_render",
            "args": { "xml_path": "...", "out_path": "..." },
            "result": { "ok": true, "pdf_path": "..." }
        }
    ],
    "debug": {
        "trace": [...],
        "usage": {
            "input_tokens": 1240,
            "output_tokens": 87
        }
    }
}
```

| Field        | Present when                               |
| ------------ | ------------------------------------------ |
| `response`   | Always                                     |
| `pdf_ref`    | Only if `fel_render` executed successfully |
| `tool_calls` | Only if MCP tools were executed            |
| `debug`      | Only if `ROUTER_DEBUG=True`                |

**Response `401`:**

```json
{ "detail": "Invalid or expired session token." }
```

**Response `429`:**

```json
{
    "detail": "Rate limit exceeded.",
    "retry_after": 43
}
```

**Response `503`:**

```json
{ "detail": "MCP server unavailable. Please try again." }
```

**Notes:**

* If `xml_ref` is present, the XML path is injected into the agent context automatically
* `history` must contain only text messages (no tool results or internal blocks)
* `X-API-Key` is used to instantiate `ChatEngine` for this request and discarded afterward

---

### `GET /render`

Downloads the PDF generated in the previous chat turn.

The file is deleted from the server after sending.

**Auth required:** `X-Preview-Token`, `Authorization: Bearer <jwt>`

**Query params:**

| Parameter | Type   | Required | Description                   |
| --------- | ------ | -------- | ----------------------------- |
| `ref`     | string | Yes      | `pdf_ref` returned by `/chat` |

**Response `200`:**

```bash
Content-Type: application/pdf
Content-Disposition: attachment; filename="factura_9E52C854_3784000961.pdf"

<binary PDF content>
```

The filename in `Content-Disposition` is the `pdf_name` from the JWT with resolved dynamic variables (`{serie}`, `{dte}`, etc.).

**Response `404`:**

```json
{ "detail": "PDF not found or already downloaded." }
```

**Response `401`:**

```json
{ "detail": "Invalid or expired session token." }
```

**Notes:**

* The PDF can only be downloaded once: it is deleted after sending
* If the JWT expired before download, the file was already removed by the cleanup task

---

### `POST /logo`

Uploads the company logo to the session’s temporary directory.

It is referenced in the visual configuration and included in the PDF.

**Auth required:** `X-Preview-Token`, `Authorization: Bearer <jwt>`

**Request:** `multipart/form-data`

| Field  | Type   | Description                    |
| ------ | ------ | ------------------------------ |
| `file` | `File` | Logo image (.jpg, .png, .webp) |

**Validations:**

* Extensions: `.jpg`, `.jpeg`, `.png`, `.webp`
* Maximum size: `MAX_LOGO_SIZE_KB` (default 200KB)

**Response `200`:**

```json
{
    "logo_ref": "logo_4c8d2f1b.jpg",
    "logo_path": "/tmp/fel/<session_uuid>/logo_4c8d2f1b.jpg"
}
```

**Notes:**

* `logo_ref` is included in `POST /session` as part of the visual configuration stored in the JWT
* If no logo is uploaded, the PDF is generated without a header logo

---

## Global error codes

| Code  | Description          | When it occurs                               |
| ----- | -------------------- | -------------------------------------------- |
| `400` | Bad Request          | Malformed input                              |
| `401` | Unauthorized         | Missing preview token or invalid/expired JWT |
| `403` | Forbidden            | Origin not allowed (CORS)                    |
| `413` | Payload Too Large    | File exceeds configured limit                |
| `422` | Unprocessable Entity | Pydantic validation failure                  |
| `429` | Too Many Requests    | Rate limit exceeded per IP                   |
| `503` | Service Unavailable  | MCP server unavailable                       |

---

## Rate limits per endpoint

| Endpoint        | Limit        | Window            |
| --------------- | ------------ | ----------------- |
| `POST /session` | 10 requests  | per minute per IP |
| `POST /upload`  | 10 requests  | per minute per IP |
| `POST /chat`    | 20 requests  | per minute per IP |
| `GET /render`   | 10 requests  | per minute per IP |
| `POST /logo`    | 10 requests  | per minute per IP |
| Global          | 100 requests | per minute per IP |

Configurable via `.env`: `RATE_LIMIT_CHAT`, `RATE_LIMIT_SESSION`.

---

## Environment variables affecting the API

| Variable              | Affected endpoint                         | Description                              |
| --------------------- | ----------------------------------------- | ---------------------------------------- |
| `PREVIEW_TOKEN`       | All                                       | System access token                      |
| `SECRET_KEY`          | `/session`, `/chat`, `/render`, `/upload` | JWT signing and verification             |
| `SESSION_TTL_MINUTES` | `/session`                                | JWT duration                             |
| `MAX_XML_SIZE_KB`     | `/upload`                                 | Max XML size                             |
| `MAX_LOGO_SIZE_KB`    | `/logo`                                   | Max logo size                            |
| `RATE_LIMIT_CHAT`     | `/chat`                                   | Requests per minute                      |
| `RATE_LIMIT_SESSION`  | `/session`                                | Requests per minute                      |
| `ROUTER_DEBUG`        | `/chat`                                   | Includes `debug` in response             |
| `ENV`                 | All                                       | Enables/disables production restrictions |

---

## Typical integration flow

```bash
1. POST /logo        → { logo_ref }
2. POST /session     → { token, session_uuid }
3. POST /upload      → { xml_ref }
4. POST /chat        → { response, pdf_ref }
5. GET  /render?ref= → <PDF binary>
```

Steps 1 and 3 are optional: the system works without a logo and without an XML (though without XML the agent cannot generate PDFs).

---

## Related files

* `technical/session-model.md` → JWT structure and session model
* `technical/engine.md` → how ChatEngine processes /chat
* `technical/mcp-server.md` → tools invoked from /chat
* `architecture/overview.md` → security model and validation layers
* `adr/0003-user-owned-api-key.md` → why X-API-Key is in header
* `adr/0005-ephemeral-link-generation.md` → JWT structure
