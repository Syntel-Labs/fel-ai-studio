# ADR 0002 — MCP as Orchestration Layer

**Status:** Accepted

**Date:** 2025-03-17

---

## Context

FEL AI Studio needs a mechanism for the AI agent (Claude) to execute concrete operations on invoices: validate totals, generate PDFs, process batches. These operations are deterministic, have well-defined inputs and outputs, and must not be freely interpreted by the model.

The design question is: how does the AI agent connect to the business logic?

In the previous code this connection existed but was not formalized.

The `ChatEngine` directly called the `McpStdioClient`, which in turn launched the server as a subprocess. It worked, but the separation of responsibilities was not explicit and the protocol was not documented.

In this version, MCP is formalized as the official orchestration layer and the decision is documented.

---

## What MCP is in this context

Model Context Protocol is an open protocol proposed by Anthropic that defines how an AI agent can discover and execute tools exposed by an external server. The protocol uses JSON-RPC 2.0 as the message format and can operate over different transports (stdio, HTTP, SSE).

In this system:

* The **MCP server** exposes FEL tools (validate, render, batch)
* The **ChatEngine** acts as an MCP client through the Anthropic SDK
* **Claude** decides when and how to use each tool via `tool_choice="auto"`

---

## Options considered

### Option A — Direct calls from FastAPI to Python functions

The `/chat` endpoint of FastAPI imports and directly calls `validateFel`, `generatePdf`, etc. The AI agent returns text with intent and FastAPI parses that intent to decide which function to call.

**Advantage:** simple, no additional processes, no intermediate protocol.

**Disadvantage:** the agent has no real visibility into the tools. FastAPI becomes a fragile intent parser. The decision logic of when to use which tool lives in ad-hoc code, not in the model. It is not extensible without modifying the router.

**Verdict:** discarded. It inverts responsibility: the code decides when to use tools instead of the model.

---

### Option B — Anthropic function calling without MCP

Tools are defined directly in the Anthropic API payload as `tools: [...]`. The model emits `tool_use` blocks, FastAPI intercepts them and executes the corresponding Python functions.

**Advantage:** simpler than MCP, no subprocess, no additional protocol. Uses Anthropic’s API in a standard way.

**Disadvantage:** tools are coupled to backend code. Adding a new tool requires modifying FastAPI. There is no separation between the agent and business logic. It does not demonstrate MCP, which is a relevant technical differentiator for the portfolio.

**Verdict:** discarded as the main architecture. It is kept as an internal mechanism of the ChatEngine to communicate with the MCP server.

---

### Option C — Separate MCP server, communication via stdio

FEL tools live in a separate process (`server_stdio.py`) that implements the MCP protocol over JSON-RPC 2.0 via stdio. The ChatEngine launches this process as a subprocess and communicates with it through the `McpStdioClient`. Claude sees the tools via standard Anthropic function calling, but the backend is decoupled.

**Advantage:** real separation between agent and business logic. The MCP server can be developed, tested, and deployed independently. Adding new tools does not require modifying the ChatEngine or FastAPI. Demonstrates a real integration of the MCP protocol as technical evidence.

**Disadvantage:** adds an additional process, stdio communication, and a protocol that must be implemented without an SDK. Harder to debug.

**Verdict:** accepted.

---

### Option D — MCP server over HTTP

The MCP server exposes tools via HTTP instead of stdio.

The `McpHttpClient` communicates with it via POST.

**Advantage:** easier to deploy independently, compatible with distributed architectures.

**Disadvantage:** adds network latency even in local deployments. Requires port management and additional configuration. For the current scope it is overengineering.

**Verdict:** discarded for v1. The `McpHttpClient` exists in the reference code and can be activated in future versions without changing the ChatEngine architecture.

---

## Decision

**MCP is the official orchestration layer of the system. FEL tools live in a separate MCP server that communicates with the ChatEngine via stdio JSON-RPC. Claude discovers and executes tools through Anthropic’s native function calling mechanism with `tool_choice="auto"`.**

---

## Resulting architecture

```bash
FastAPI /chat
    │
    ▼
ChatEngine (per request)
    ├── Anthropic Client → Claude (tool_choice="auto")
    │       │
    │       │  tool_use blocks
    │       ▼
    └── McpStdioClient
            │  JSON-RPC over stdio
            ▼
        server_stdio.py (separate process)
            ├── fel_validate → validateFel()
            ├── fel_render   → generatePdf()
            └── fel_batch    → batchFel()
```

---

## Tool-use loop

```bash
1. ChatEngine sends message + tools catalog to Claude
2. Claude responds with a tool_use block
   └── { name: "fel_render", input: { xml_path, out_path, config } }
3. ChatEngine intercepts the block
4. Sanitizes args (sandbox path by session_uuid)
5. Calls McpStdioClient.callTool(name, args)
6. McpStdioClient sends JSON-RPC to the MCP server
7. Server executes the function and returns result
8. ChatEngine builds tool_result and reinjects it into the context
9. Claude receives the result and generates a natural language response
10. ChatEngine returns finalText to the FastAPI endpoint
```

Maximum configured hops (`maxHops=3`) to avoid infinite loops.

---

## Behavior when validation returns `ok=False`

**Decision: Variant 1 — Claude decides.**

The result of `fel_validate` is returned to the model as `tool_result`. Claude evaluates the issues and decides whether to continue with `fel_render` or inform the user. The model acts as the arbiter.

Reason: less coupling in the MCP server, more flexibility in v1. If cases are detected where the model consistently makes incorrect decisions, it will migrate to Variant 2 (MCP server blocks if `ok=False`) through a new ADR without changing the overall architecture.

---

## Consequences

**Positive:**

* The MCP server can be tested in isolation with direct JSON-RPC
* Adding new tools does not affect FastAPI or the ChatEngine
* The ChatEngine is tool-agnostic: it discovers them at runtime
* Demonstrates a real MCP implementation without SDK as technical evidence
* The same MCP server is distributed as a standalone Docker image (see ADR 0006)

**Negative:**

* The MCP server runs as a subprocess: if it crashes, the ChatEngine fails
* stdio communication is harder to debug than HTTP
* The JSON-RPC protocol must be implemented and maintained manually
* Claude may decide to generate a PDF with minor validation issues

**Neutral:**

* The existing `McpHttpClient` can be enabled in v2 without engine changes
* RPC logs (`rpc_logger.py`) are essential for debugging

---

## Constraints imposed by this decision

* Every new tool must be implemented in the MCP server, not in FastAPI
* The ChatEngine cannot call business Python functions directly
* The sandbox path must be verified by `session_uuid` before each tool call
* The MCP server must be started before FastAPI accepts requests

---

## Future review

If the system scales to multiple MCP servers (FEL, accounting, digital signature), migrating from stdio to HTTP as transport will be evaluated. This change does not affect the ChatEngine or FastAPI, only the active MCP client.

---

## Related files

* `adr/0001-stateless-session-model.md` → overall architecture context
* `adr/0006-docker-hub-mcp-distribution.md` → standalone MCP distribution
* `technical/engine.md` → implementation of the tool-use loop
* `technical/mcp-server.md` → protocol and exposed tools
* `architecture/overview.md` → complete component diagram
