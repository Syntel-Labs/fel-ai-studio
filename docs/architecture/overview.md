# Architecture Overview — FEL AI Studio

## Design principle

The system is designed to be completely stateless from the server’s perspective. There is no user database, no persistent sessions, no permanent invoice storage. All state lives in the client or in temporary files with bounded TTL.

Security does not rely on obscurity — the code is public. It relies on explicit layers of protection at every entry point.

---

## Two operating modes

### Web mode

Full application: React + Vite → FastAPI → ChatEngine → MCP Server.

The user accesses with `PREVIEW_TOKEN`, configures visual identity, attaches the XML in the chat, and downloads the PDF.

### Docker Hub mode (MCP standalone)

Only the MCP server packaged as a public Docker image. It connects to Claude Desktop via `settings`. The user provides `MCP_TOKEN` and `ANTHROPIC_API_KEY`. There is no frontend or FastAPI.

---

## Main components

```bash
Browser (React + Vite)
│
├── Configuration panel     → 7 color tokens, logo, PDF name,
│                            API Key (sessionStorage), live preview
├── Link generator          → serializes config into temporary JWT
└── Chat UI                 → attaches XML, interaction with agent
        │
        ▼
FastAPI (Backend)
│
├── /session                → generates and validates session tokens
├── /upload                 → receives XML, stores it in /tmp/fel/<session>/<uuid>.xml
├── /chat                   → receives messages + XML reference
└── /render                 → download endpoint for generated PDF
        │
        ▼
ChatEngine (Python)
│
├── Anthropic client        → Claude as main agent
├── tool_choice="auto"      → model decides when to use tools
└── McpStdioClient          → communicates with MCP server
        │
        ▼
MCP Server (stdio, no SDK)
│
├── fel_validate            → validates VAT and total consistency
├── fel_render              → generates PDF with ReportLab + PdfConfig
└── fel_batch               → processes directory of XMLs
        │
        ▼
ReportLab
└── Final PDF               → direct download, deleted after sending
```

---

## Full session flow (web mode)

```bash
1. User accesses preview
   └── Presents PREVIEW_TOKEN → 401 if invalid

2. Configures visual identity in the panel:
   ├── 7 color tokens (primary, primary_text, secondary,
   │   secondary_text, text_main, text_soft, background)
   ├── Logo (base64 in sessionStorage)
   ├── PDF name (e.g., factura_{serie}_{dte}.pdf)
   └── Everything in sessionStorage, never sent to the server

3. Generates session link
   └── POST /session → JWT signed with visual config

4. Opens chat with active link
5. Attaches invoice XML as file
   └── POST /upload → stored in /tmp/fel/<session_uuid>/<file_uuid>.xml
   └── Session-isolated: two users never share paths

6. Types: "generate the PDF of this invoice"
7. ChatEngine receives message + xml_path from session context
   ├── Calls Claude with tool_choice="auto"
   ├── Claude emits tool_use → fel_render
   └── ChatEngine executes via MCP with PdfConfig from JWT

8. MCP Server generates PDF with ReportLab
9. PDF available at /render → direct download
   └── File deleted after sending

10. Automatic cleanup:
    └── XML TTL = JWT duration (default 60 min)
    └── Background task deletes expired session files
```

---

## Docker Hub mode flow (Claude Desktop)

```bash
1. User downloads image: docker pull felai/mcp-server
2. Configures in Claude Desktop settings:
   MCP_TOKEN=<secret>
   ANTHROPIC_API_KEY=<their key>
3. Claude Desktop launches container as MCP server
4. Claude sees tools: fel_validate, fel_render, fel_batch
5. User attaches XML in Claude Desktop
6. Claude orchestrates tools directly
```

---

## XML handling — session isolation

The XML is never processed by the AI. The AI only receives the path.

```bash
/tmp/fel/
├── <session_uuid_1>/
│   └── <file_uuid>.xml     ← user A
├── <session_uuid_2>/
│   └── <file_uuid>.xml     ← user B
└── ...
```

* Each session has its own subdirectory with UUID v4
* Two simultaneous uploads never share paths nor are accessible between each other by the MCP server (session-based path sandboxing)
* TTL: directories are deleted when the JWT expires
* The original filename is discarded on upload

---

## Visual customization — 7 color tokens

| Token            | Applied to                         |
| ---------------- | ---------------------------------- |
| `primary`        | Top bar background, table header   |
| `primary_text`   | Text in top bar, table header text |
| `secondary`      | Footer background                  |
| `secondary_text` | Footer icons, footer text          |
| `text_main`      | Bold labels, Total row (emphasis)  |
| `text_soft`      | Secondary values, table rows       |
| `background`     | Document background                |

The configuration panel shows a real-time preview of the PDF with current colors before generating the link.

---

## Key architectural decisions

| Decision                          | Rationale                                         |
| --------------------------------- | ------------------------------------------------- |
| MCP as orchestration layer        | Separates business logic from AI agent            |
| stdio JSON-RPC without SDK        | Full control of protocol, no dependencies         |
| XML in /tmp isolated per session  | User privacy, no data mixing                      |
| AI does not touch XML             | Model orchestrates, MCP server processes          |
| API Key on client                 | No operational cost, no leakage risk              |
| sessionStorage (not localStorage) | Config cleared when tab closes                    |
| FastAPI as backend                | Async, lightweight, ideal for AI streaming        |
| Docker Hub MCP standalone         | Enables personal use without web deployment       |
| Docker + docker-compose           | Reproducibility from first clone                  |
| No database                       | Simplifies deployment, reinforces privacy promise |
| PREVIEW_TOKEN access              | Controls system usage in preview mode             |

---

## Security model

Layers are applied in order. A request that fails does not proceed to the next.

```bash
Incoming request
      │
      ▼
[ Layer 1 ] Valid PREVIEW_TOKEN?        → 401 if not
      │
      ▼
[ Layer 2 ] HTTPS in production?        → reject HTTP if ENV=production
      │
      ▼
[ Layer 3 ] Origin in CORS_ORIGINS?     → 403 if not
      │
      ▼
[ Layer 4 ] Rate limit per IP OK?       → 429 if exceeded
      │
      ▼
[ Layer 5 ] Valid and sanitized input?  → 422 if not
      │
      ▼
  Normal processing
```

Full detail of each layer in `architecture/overview.md` section “Security model” — unchanged from the previous version.

---

## Environment variables

| Variable              | Description                 | Default                     |
| --------------------- | --------------------------- | --------------------------- |
| `PREVIEW_TOKEN`       | Access to web system        | — (required in prod)        |
| `MCP_TOKEN`           | Access to Docker MCP server | — (required)                |
| `SECRET_KEY`          | JWT signing                 | — (required in prod)        |
| `CORS_ORIGINS`        | Allowed origins             | `http://localhost:5173`     |
| `ENV`                 | Runtime environment         | `development`               |
| `HTTPS_ONLY`          | Enforce HTTPS               | `false` in dev              |
| `RATE_LIMIT_CHAT`     | Limit on /chat              | `20/minute`                 |
| `RATE_LIMIT_SESSION`  | Limit on /session           | `10/minute`                 |
| `MAX_XML_SIZE_KB`     | Max XML size                | `500`                       |
| `SESSION_TTL_MINUTES` | JWT and XML TTL             | `60`                        |
| `ALLOWED_ROOTS`       | Allowed paths for MCP       | `data/xml,data/out`         |
| `PDF_NAME_TEMPLATE`   | Generated PDF name          | `factura_{serie}_{dte}.pdf` |

---

## Related files

* `adr/0001-stateless-session-model.md` → stateless session model
* `adr/0002-mcp-as-orchestration-layer.md` → why MCP instead of direct calls
* `adr/0003-user-owned-api-key.md` → why API Key never touches the server
* `adr/0005-ephemeral-link-generation.md` → JWT construction
* `technical/engine.md` → ChatEngine and tool-use loop
* `technical/mcp-server.md` → JSON-RPC protocol and tools
* `technical/session-model.md` → session model and XML isolation
* `technical/pdf-generation.md` → PdfConfig and color tokens
* `architecture/stack.md` → justification of each technology
* `architecture/flow-diagram.md` → Mermaid diagrams of full flow
