# Product Vision — FEL AI Studio

## Problem

Electronic FEL invoices issued by Guatemala’s SAT are rigid documents with no visual identity, failing to reflect any company’s brand.

Transforming them into presentable documents is a manual, repetitive, and error-prone process.

## Solution

FEL AI Studio is a web platform that converts FEL invoices (XML) into professional, customized PDFs, orchestrated by AI through the MCP protocol.

The user configures their visual identity once, generates a session link, attaches their XML directly in the chat, and downloads the customized invoice without the system permanently storing any sensitive data.

## Value proposition

* No invoice database: processing is ephemeral, temporary files are automatically deleted
* No AI operational cost for the provider: the user uses their own Anthropic API Key
* Real AI integration: not just a PDF generator, but an agent that understands natural language requests and executes MCP tools
* Full visual identity: 7 semantic color tokens applied to each section of the document (header bar, table, footer, text, background)
* Two usage modes: web (preview + chat) and Docker Hub (MCP server for Claude Desktop)
* Demonstrable and extensible: clean architecture, ready to scale into SaaS

## Target users

**Potential customer (Guatemalan SMB):**

A company issuing FEL invoices that wants to deliver them with its brand identity, without relying on an accountant or designer for each document.

**Recruiter / tech company:**

Concrete evidence of AI + backend + MCP protocol integration + document generation in a modern, well-documented stack.

**Technical community (GitHub):**

Reference implementation of MCP without SDK, Claude orchestration, and stateless design applied to a real Latin American problem.

## Two distribution modes

### Web mode (portfolio / preview)

Full web application with configuration panel, AI chat, and PDF download. Requires `PREVIEW_TOKEN` for access. The user provides their own Anthropic API Key.

### Docker Hub mode (personal use / Claude Desktop)

Public MCP server image exposing `fel_validate`, `fel_render`, and `fel_batch` as tools. Connects to Claude Desktop by adding the container in `settings`. Requires `MCP_TOKEN` and `ANTHROPIC_API_KEY` as environment variables. No frontend or FastAPI included.

## Stack

| Layer            | Technology                           |
| ---------------- | ------------------------------------ |
| Backend / API    | FastAPI + Python                     |
| MCP Server       | Python (stdio JSON-RPC, no SDK)      |
| PDF Engine       | ReportLab                            |
| AI / LLM         | Claude (Anthropic) via user API      |
| Frontend         | React + Vite                         |
| Infrastructure   | Docker + docker-compose              |
| MCP Distribution | Docker Hub (public image with token) |

## What this project is not

* Not an accounting system
* Not a replacement for the SAT portal
* Does not validate invoices with SAT (only internal consistency: VAT, totals)
* Does not store invoices, users, or API Keys permanently
* Does not process XML with AI: the AI orchestrates, the MCP server processes

## Future extensions

* Multi-company with saved templates
* Usage dashboard and metrics
* Public API for integration with accounting systems
* SaaS model with plans and billing

## License and project usage

FEL AI Studio is a personal portfolio project. The code and documentation are public for technical demonstration and learning purposes.

Not allowed:

* Using the code as the base of a commercial product
* Redistributing it as your own work
* Financially exploiting it without explicit authorization from the author

Allowed:

* Reading, studying, and citing it as a reference
* Mentioning it with attribution to the original author

## Current status

Portfolio phase — functional preview.

The MCP engine and PDF generation exist as an academic proof of concept.

This version refactors toward ephemeral sessions, UI-based configuration, semantic color palette, and a production-ready architecture.
