# Stack — FEL AI Studio

## Selection criteria

Each technology in the stack was chosen based on these criteria, in order:

1. Appropriate for the specific problem it solves
2. Standard in the applied AI and modern backend ecosystem
3. Recognizable to recruiters and the technical community
4. No unnecessary dependencies that increase deployment complexity

---

## Overview

```bash id="wz7h3u"
┌─────────────────────────────────────────────────────┐
│                    FRONTEND                         │
│              React 18 + Vite + Tailwind             │
│                   Lucide Icons                      │
└─────────────────────┬───────────────────────────────┘
                      │ HTTP / REST
┌─────────────────────▼───────────────────────────────┐
│                    BACKEND                          │
│                   FastAPI                           │
│              slowapi (rate limiting)                │
│               python-jose (JWT)                     │
└──────────┬──────────────────────┬───────────────────┘
           │ subprocess stdio     │ StreamingResponse
┌──────────▼──────────┐  ┌────────▼───────────────────┐
│    MCP SERVER       │  │    TEMP FILES              │
│  server_stdio.py    │  │   /tmp/fel/<session_uuid>/ │
│  JSON-RPC 2.0       │  │   XML + PDF (TTL = JWT)    │
│  no SDK             │  └────────────────────────────┘
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│   PDF ENGINE        │
│   ReportLab         │
│   ElementTree       │
│   qrcode            │
└─────────────────────┘
```

---

## Frontend

### React 18 + Vite

**Role:** user interface — configuration panel, chat, PDF preview.

**Why React:**

Industry standard for SPAs. The configuration panel requires real-time reactive state (color preview, logo, PDF name updating as the user edits). React handles this naturally without overengineering.

**Why Vite:**

Modern build tool, instant HMR in development, optimized production output. Replaces Create React App without its limitations.

---

### Tailwind CSS

**Role:** UI styling.

**Why Tailwind:**

Utility-first approach enables building custom interfaces without a third-party design system. FEL AI Studio’s design is intentional and differentiating — a system like MUI would hide those decisions behind generic components.

---

### Lucide Icons

**Role:** interface iconography.

**Why Lucide:**

Tree-shakeable: only used icons are included. No local assets, no SVGs in the repo (consistent with ADR 0004). Clean API as React components.

---

### Google Fonts (CDN)

**Role:** web typography.

**Why CDN:**

No TTF files in the repo. Loaded on first request and cached. Consistent with ADR 0004.

---

## Backend

### FastAPI

**Role:** REST API — endpoints `/session`, `/upload`, `/chat`, `/render`.

**Why FastAPI:**

* Native async: ideal for proxying AI requests that may take seconds
* Typed with Pydantic: automatic input validation, clear schemas
* `StreamingResponse`: streams the PDF without loading it fully into memory
* `BackgroundTasks`: cleans temporary files without blocking responses
* Standard in the Python AI ecosystem

---

### python-jose

**Role:** generation and verification of session JWTs.

**Why python-jose:**

Mature JWT implementation in Python. Native HS256 support. No heavy dependencies.

---

### slowapi

**Role:** IP-based rate limiting on critical endpoints.

**Why slowapi:**

Wrapper around `limits` designed for FastAPI. Decorator-based integration, no complex middleware. Per-endpoint configuration.

---

### python-dotenv

**Role:** loading environment variables from `.env`.

**Why:** de facto standard in Python projects. No alternatives provide meaningful added value here.

---

## MCP Server

### Python (no SDK)

**Role:** MCP server exposing `fel_validate`, `fel_render`, `fel_batch` via JSON-RPC 2.0 over stdio.

**Why no SDK:**

The MCP protocol over stdio is simple enough to implement directly with `sys.stdin` / `sys.stdout` + `json`. Removing the SDK gives full control of the protocol, no forced updates, and no dependency risk.

It is also a strong technical signal demonstrating protocol-level understanding (relevant for a portfolio).

---

### subprocess + queue + threading

**Role:** MCP client in `ChatEngine` that launches and communicates with the MCP server as a child process.

**Why stdlib:**

`subprocess.Popen` + `queue.Queue` + `threading.Thread` solve the problem directly: spawn a process, read stdout asynchronously, and perform RPC without blocking. No need for external dependencies.

---

## PDF Engine

### ReportLab

**Role:** programmatic PDF generation with custom layout.

**Why ReportLab:**

* Pixel-perfect layout control: required for accurate FEL invoice structure
* Direct canvas access: precise positioning of elements
* `HexColor`: maps user color tokens directly
* Mature, stable, lightweight
* Alternatives like WeasyPrint or xhtml2pdf require HTML/CSS as an intermediate layer, adding unnecessary complexity

---

### ElementTree (stdlib)

**Role:** parsing FEL XML.

**Why stdlib:**

FEL XML is well-structured with known namespaces. `ElementTree` handles it without external dependencies. `lxml` would be overengineering.

---

### qrcode

**Role:** generating QR code with SAT authorization number.

**Why qrcode:**

Lightweight, purpose-built library. Generates images directly embeddable in ReportLab. No better alternative for this use case.

---

## AI / LLM

### Claude (Anthropic) — user API

**Role:** main agent that interprets natural language and decides when/how to invoke MCP tools.

**Why Claude:**

`tool_choice="auto"` with `tool_use` blocks is Anthropic’s native orchestration mechanism. The system is built around this. Changing models would require modifying ChatEngine.

**Why user API:**

Zero operational cost for the system. Users control their own usage. Aligned with ADR 0003.

---

### anthropic (Python SDK)

**Role:** official Anthropic client in `ChatEngine`.

**Why official SDK:**

Handles authentication, retries, streaming, and `tool_use` / `tool_result` formats reliably. No advantage in building a custom HTTP client for this layer.

---

## Infrastructure

### Docker + docker-compose

**Role:** packaging and local orchestration of the full system.

**Why Docker:**

Guaranteed reproducibility: `docker-compose up` runs everything without installing Python, Node, or dependencies locally. Standard for portfolio projects evaluated by recruiters or the technical community.

**Services in docker-compose:**

| Service      | Base image                   | Port |
| ------------ | ---------------------------- | ---- |
| `frontend`   | node:20-alpine               | 5173 |
| `backend`    | python:3.12-slim             | 8000 |
| `mcp-server` | — (child process of backend) | —    |

The MCP server is not a standalone service in docker-compose:
it runs as a subprocess of the backend via `McpStdioClient`.

---

### Docker Hub (standalone MCP image)

**Role:** distribution of MCP server for personal use with Claude Desktop.

**Why Docker Hub:**

Public distribution with a single `docker pull`. No installation friction for personal use (ADR 0006).

---

### GitHub Actions

**Role:** CI for automatic build and push of MCP image to Docker Hub on each release.

**Why GitHub Actions:**

Native to GitHub, no external services. Simple workflow: tag → build → push.

---

## Stack summary

| Layer              | Technology       | Target version |
| ------------------ | ---------------- | -------------- |
| Frontend framework | React            | 18             |
| Frontend build     | Vite             | 5              |
| Frontend styles    | Tailwind CSS     | 3              |
| Frontend icons     | Lucide React     | latest         |
| Backend API        | FastAPI          | 0.110+         |
| Backend validation | Pydantic         | v2             |
| Backend JWT        | python-jose      | 3.3+           |
| Backend rate limit | slowapi          | 0.5+           |
| MCP server         | Python stdlib    | 3.12           |
| PDF engine         | ReportLab        | 4+             |
| XML parser         | ElementTree      | stdlib         |
| QR code            | qrcode           | 7+             |
| AI client          | anthropic SDK    | 0.25+          |
| Containers         | Docker + compose | 25+ / 2.24+    |
| CI/CD              | GitHub Actions   | —              |

---

## What is not in the stack and why

| Technology          | Reason for exclusion                               |
| ------------------- | -------------------------------------------------- |
| Redis               | No server-side state (ADR 0001)                    |
| PostgreSQL / SQLite | No persistence of users or invoices                |
| Celery / workers    | No long-running background processing              |
| Next.js             | SSR adds no value in a token-authenticated SPA     |
| MUI / Chakra        | Design system hides intentional UI decisions       |
| lxml                | ElementTree solves the use case without extra deps |
| WeasyPrint          | ReportLab gives direct layout control              |
| Official MCP SDK    | Full protocol control via stdlib (ADR 0002)        |

---

## Related files

* `adr/0002-mcp-as-orchestration-layer.md` → why MCP without SDK
* `adr/0004-no-local-fonts-or-icons.md` → Lucide and Google Fonts
* `adr/0006-docker-hub-mcp-distribution.md` → standalone Docker image
* `architecture/overview.md` → how components connect
* `architecture/flow-diagram.md` → full flow using this stack
