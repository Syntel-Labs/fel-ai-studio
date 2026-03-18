# ADR 0006 — Docker Hub MCP Distribution

**Status:** Accepted

**Date:** 2025-03-17

---

## Context

FEL AI Studio has two technical audiences with different needs:

1. **Web user:** accesses the full system via browser, with configuration panel, chat, and PDF download
2. **Technical / personal user:** wants to use FEL tools directly from Claude Desktop or another MCP client, without deploying the full web application

The second case requires a way to distribute only the MCP server so that anyone can connect it to their MCP client with minimal setup. Since the code is public on GitHub, distribution as a Docker image on Docker Hub is the most straightforward approach.

---

## Options considered — Distribution

### Option A — Only document manual installation

The user clones the repo, installs Python dependencies, and runs
`server_stdio.py` directly.

**Advantage:** no additional infrastructure.

**Disadvantage:** requires Python, dependencies, and knowledge of the repo. Installation friction is high for personal use. It is not a real distribution—just instructions.

**Verdict:** discarded. Does not solve the distribution problem.

---

### Option B — Public Docker Hub image with token ✓

Public image on Docker Hub containing only the MCP server.

Access to the server requires `MCP_TOKEN` as an environment variable.

Without the token, the server rejects requests.

**Advantage:** installation with a single `docker pull`. Compatible with Claude Desktop and any MCP client supporting stdio. The token protects against unauthorized use without requiring a private image.

**Disadvantage:** anyone can download the image even if they cannot use it without the token. The image is public: the server code is visible, consistent with the public repo.

**Verdict:** accepted.

---

### Option C — Private Docker Hub image

Private image requiring Docker Hub login to download.

**Advantage:** control over who can download.

**Disadvantage:** adds authentication friction. Contradicts the goal of easy distribution for personal use. A private image in a public repo is inconsistent.

**Verdict:** discarded.

---

## Options considered — Visual customization in Docker mode

In web mode, customization comes from the JWT built in the configuration panel. In Docker mode there is no panel or JWT, so visual configuration must be handled differently.

### Option A — Environment variables per parameter

Each color token and visual parameter is passed as an individual environment variable in `docker run`:

```bash
-e FEL_COLOR_PRIMARY=#fe5101
-e FEL_COLOR_PRIMARY_TEXT=#ffffff
-e FEL_COLOR_SECONDARY=#343534
...
```

**Advantage:** no additional files, everything in the startup command.

**Disadvantage:** the `docker run` command becomes very long with 7 colors plus logo, PDF name, and other parameters. Hard to maintain and error-prone.

**Verdict:** discarded as the main mechanism.

---

### Option B — `fel_config.json` file in mounted volume ✓

The user places a `fel_config.json` file inside the volume that is already required for XMLs and PDFs. The server reads it at startup and builds `PdfConfig` from it.

**Advantage:** a single editable, readable, locally versionable file. Keeps the startup command clean. The user can have different `fel_config.json` files for different companies simply by changing the mounted volume.

**Disadvantage:** the user must create the file before using the server. If the file does not exist, the server must have a clear behavior (use defaults or reject startup).

**Verdict:** accepted as the main mechanism.

---

### Option C — Arguments in Claude prompt

The user specifies colors and logo directly to Claude in each conversation. Claude passes those parameters as arguments to `fel_render`.

**Advantage:** maximum per-conversation flexibility, no prior files needed.

**Disadvantage:** the user must repeat configuration in every session. Not ergonomic for regular use. Visual parameters should not be the user prompt’s responsibility.

**Verdict:** discarded as the main mechanism. Kept as a one-off override over file configuration.

---

## Decision

**The MCP server is distributed as a public Docker Hub image. Access requires `MCP_TOKEN` as an environment variable. Visual customization is configured via a `fel_config.json` file placed by the user in the mounted volume. If the file does not exist, the server uses default values. Arguments in the Claude prompt can override individual parameters when needed.**

---

## Precedence hierarchy for PdfConfig in Docker mode

```bash
explicit arguments in the tool call (from Claude prompt)
        │  overrides
        ▼
fel_config.json in mounted volume
        │  overrides
        ▼
container environment variables (MCP_TOKEN required,
        │  others optional as fallback)
        ▼
system default values
```

---

## What the image includes

```bash
felai/mcp-server (Docker Hub)
│
├── server_stdio.py     → MCP server (JSON-RPC stdio, no SDK)
├── rpc_logger.py       → JSON-RPC message logger
├── fel_pdf.py          → PDF generation with ReportLab
├── config.py           → configurable parameters via env and file
└── requirements.txt    → ReportLab + minimal dependencies
```

Does not include:

* FastAPI or any HTTP server
* ChatEngine or Anthropic client
* React frontend
* Session logic or JWT

---

## Mounted volume structure

```bash
data/                        ← root of mounted volume
├── fel_config.json          ← visual configuration (required for customization)
├── xml/                     ← input XMLs
├── out/                     ← generated PDFs
└── logos/
    └── logo.jpg             ← logo referenced in fel_config.json
```

---

## `fel_config.json` structure

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
  "logo_path": "/app/data/logos/logo.jpg",
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

If `fel_config.json` does not exist in the volume, the server starts with system default values and indicates this in the RPC logs.

It does not fail: it degrades gracefully to the base configuration.

---

## Configuration for Claude Desktop

The user adds to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "fel-ai-studio": {
      "command": "docker",
      "args": [
        "run", "--rm", "-i",
        "-e", "MCP_TOKEN=<your_token>",
        "-v", "/local/path/data:/app/data",
        "felai/mcp-server:latest"
      ]
    }
  }
}
```

The user places their `fel_config.json` and logo in `/local/path/data/` before using the server. Claude Desktop launches the container as a stdio process and has access to `fel_validate`, `fel_render`, and `fel_batch`.

---

## Token authentication model

```bash
Incoming JSON-RPC request
  └── params.mcp_token present?
        ├── Yes, matches MCP_TOKEN → process normally
        └── No / mismatch → JSON-RPC error -32001 "Unauthorized"
```

* `MCP_TOKEN` is configured as a container environment variable
* The token is compared using `secrets.compare_digest` (timing-safe)
* Without `MCP_TOKEN` configured, the server does not start

---

## Image versioning

| Tag      | Description               |
| -------- | ------------------------- |
| `latest` | Latest stable version     |
| `v1.0.0` | Specific semantic version |

Versioning follows the main GitHub repository.

Each GitHub release triggers an automatic build and push to Docker Hub via GitHub Actions.

---

## Consequences

**Positive:**

* Installation with a single command, no local Python dependencies
* Compatible with Claude Desktop and any stdio MCP client
* The author can use the system personally without deploying the web app
* Visual configuration lives in an editable, versionable file
* Different companies = different volumes with different `fel_config.json`
* Demonstrates ability to distribute MCP tools as a product

**Negative:**

* Requires Docker installed on the user’s machine
* The user must create `fel_config.json` before customization
* `MCP_TOKEN` must be kept secret by the user
* No visual preview: the PDF is written directly to the volume

**Neutral:**

* The image is public: code is visible, consistent with the repo
* Claude prompt arguments allow one-off overrides without editing the config file

---

## Constraints imposed by this decision

* The image must never include credentials, tokens, or API Keys hardcoded
* The server does not start without `MCP_TOKEN` configured
* `fel_config.json` is never included in the image: it always comes from the volume
* The MCP server `Dockerfile` must remain separate from the web application `Dockerfile`
* The server must validate `fel_config.json` at startup and log warnings if fields are missing (never fail silently)

---

## Future review

If support for additional tools is added (digital signature, email sending, accounting integration), they are published as new versions of the same image. No new ADR is required unless the distribution, authentication, or visual configuration model changes.

---

## Related files

* `adr/0002-mcp-as-orchestration-layer.md` → MCP server architecture
* `adr/0003-user-owned-api-key.md` → API Key model in Docker mode
* `technical/mcp-server.md` → protocol and exposed tools
* `technical/pdf-generation.md` → how PdfConfig is built in Docker mode
* `architecture/overview.md` → two system operation modes
