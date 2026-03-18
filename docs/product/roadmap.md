# Roadmap — FEL AI Studio

## Prioritization criteria

Features are prioritized based on these criteria, in order:

1. **Demonstrates technical value** — provides verifiable portfolio evidence or differentiates the product from a generic PDF generator
2. **Solves a real problem** — has a concrete user who needs it (Rodrigo, Valeria, or Diego per `product/personas.md`)
3. **Does not break the privacy promise** — any feature requiring user data persistence must first supersede ADR 0001

---

## Current state — v0.1.0 (academic base)

Starting point before this rewrite. Exists as a proof of concept.

**What works:**

* MCP server over stdio without SDK (`fel_validate`, `fel_render`, `fel_batch`)
* ChatEngine with tool-use loop and `tool_choice="auto"`
* PDF generation with ReportLab (colors hardcoded from `.env`)
* CLI frontend with rich + REPL
* Configuration exclusively via `.env`

**What does not exist yet:**

* Web frontend
* JWT-based session model
* XML isolation per user
* Visual configuration panel
* Docker Hub distribution
* Security (rate limiting, CORS, PREVIEW_TOKEN)

---

## v1.0.0 — Functional portfolio

**Goal:** complete, demonstrable system. A recruiter can clone it, run `docker-compose up`, and see the end-to-end flow in under 5 minutes.

### Backend

* [ ] FastAPI with endpoints `/session`, `/upload`, `/chat`, `/render`, `/logo`
* [ ] JWT session model (HS256, configurable TTL)
* [ ] XML isolation by `session_uuid` in `/tmp/fel/`
* [ ] Automatic cleanup of temporary files via background task
* [ ] Rate limiting with `slowapi`
* [ ] CORS restricted to configured domains
* [ ] PREVIEW_TOKEN as access control
* [ ] Input validation and sanitization (XML, logo, paths)
* [ ] `PdfConfig` built from JWT (7 color tokens)
* [ ] Configurable PDF name with dynamic variables

### MCP Server

* [ ] Clean rewrite of `server_stdio.py` with required `out_path`
* [ ] Support for `colors` as argument in `fel_render`
* [ ] `MCP_TOKEN` for authentication in Docker mode
* [ ] Read `fel_config.json` from mounted volume
* [ ] Precedence hierarchy: args > config.json > env > defaults
* [ ] Update `protocolVersion` to `2025-06-18`

### PDF Engine

* [ ] `PdfConfig` as dataclass with 7 semantic tokens
* [ ] Signature `generatePdf(xmlPath, outputPdf, config: PdfConfig)`
* [ ] QR code using `qrcode` library
* [ ] Helvetica as base font (no local TTF assets)
* [ ] Footer icons as Unicode characters
* [ ] PDF name resolved with dynamic variables

### Frontend

* [ ] React 18 + Vite + Tailwind + Lucide
* [ ] Sidebar with three sections: HOME, CONFIG, CHAT
* [ ] Configuration panel with real-time preview
* [ ] Selector for 7 color tokens with semantic naming
* [ ] Logo upload with drag & drop
* [ ] Link generation with expiration countdown
* [ ] Chat with XML file attachment
* [ ] Inline download button in agent message
* [ ] Process view and logs view as optional tabs
* [ ] `sessionStorage` rehydration on reload

### Infrastructure

* [ ] `Dockerfile` for backend (python:3.12-slim)
* [ ] `Dockerfile` for frontend (node:20-alpine)
* [ ] Separate `Dockerfile` for standalone MCP server
* [ ] `docker-compose.yml` with `frontend` and `backend` services
* [ ] `.env.example` with all variables documented
* [ ] GitHub Actions: build and push MCP image to Docker Hub on each release

### Documentation

* [ ] Main README with installation and usage instructions
* [ ] Complete `docs/` following defined phase order
* [ ] `LICENSE` with commercial use restrictions

---

## v1.1.0 — Portfolio polish

**Goal:** remove friction for all three user profiles. No architectural changes.

* [ ] README with GIF or video demo of full flow
* [ ] Landing page in HOME with simplified architecture diagram
* [ ] Dark mode in frontend
* [ ] Improved PDF preview: closer to real document
* [ ] More descriptive error messages in chat
* [ ] Progress indicator during PDF generation
* [ ] Batch support from chat with ZIP download

---

## v1.2.0 — Public Docker Hub

**Goal:** MCP image is published, documented, and usable from Claude Desktop with minimal friction.

* [ ] `felai/mcp-server` image published on Docker Hub
* [ ] Dedicated Docker Hub README
* [ ] Claude Desktop configuration documentation in repo
* [ ] `fel_config.json` with JSON schema validated at startup
* [ ] Clear log warnings when defaults are used
* [ ] Support for multiple volumes for local multi-company setups

---

## v2.0.0 — SaaS product (requires superseding ADRs 0001 and 0003)

**Goal:** transform the portfolio into a real product with users, accounts, and a business model. This version requires architectural decisions that contradict the current stateless model and must be documented in new ADRs before implementation.

**ADRs superseded:**

* ADR 0001 → add user database and persistent sessions
* ADR 0003 → operator manages API keys with encryption at rest

### Authentication and accounts

* [ ] Account system (email + password or SSO)
* [ ] API key management per account with encryption at rest
* [ ] Invoice history per account
* [ ] Usage dashboard and token consumption tracking

### Multi-company

* [ ] Saved templates per company
* [ ] Multiple visual configurations per account
* [ ] Custom subdomain per company

### Business model

* [ ] Free / Pro / Business plans
* [ ] Usage limits per plan
* [ ] Integrated billing (Stripe or similar)
* [ ] Public API with client API key authentication

### Integrations

* [ ] Webhook to receive XMLs from accounting systems
* [ ] Integration with Guatemalan FEL providers (Infile, Digifact, etc.)
* [ ] Export to multiple formats (PDF, HTML, CSV)
* [ ] Direct email delivery to invoice recipient

---

## Version-independent technical extensions

Features that can be implemented in any version without major architectural changes:

| Extension                                                   | Minimum version | Impact                     |
| ----------------------------------------------------------- | --------------- | -------------------------- |
| Custom fonts via configuration                              | v1.1.0          | Extends PdfConfig, new ADR |
| PDF digital signature                                       | v1.2.0          | New MCP tool               |
| Support for more DTE types (special invoices, credit notes) | v1.1.0          | Extends readFelXml         |
| Validation against official SAT XSD schema                  | v1.1.0          | Extends fel_validate       |
| MCP server over HTTP (McpHttpClient active)                 | v2.0.0          | Activates existing code    |
| Automated tests for MCP server                              | v1.0.0          | No architectural changes   |
| E2E tests for web flow                                      | v1.1.0          | No architectural changes   |

---

## What is not in the roadmap and why

| Feature                | Reason for exclusion                                                               |
| ---------------------- | ---------------------------------------------------------------------------------- |
| Native mobile app      | Web frontend is responsive. Native app duplicates effort without added value in v1 |
| OCR for invoice images | Outside FEL domain. FEL invoices are always XML                                    |
| FEL XML generation     | Requires SAT integration. Out of scope                                             |
| Cloud PDF storage      | Contradicts privacy promise until v2.0.0                                           |
| Multi-language (i18n)  | Domain is Guatemala. Spanish is the only relevant language in v1                   |

---

## Related files

* `product/vision.md` → value proposition this roadmap implements
* `product/personas.md` → personas driving prioritization
* `adr/0001-stateless-session-model.md` → constraint v2.0.0 supersedes
* `adr/0003-user-owned-api-key.md` → constraint v2.0.0 supersedes
* `progress/changelog.md` → record of implemented features
