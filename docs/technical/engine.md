# Engine — FEL AI Studio

## Responsibility

`ChatEngine` is the orchestration layer between the AI agent (Claude) and the MCP server. It receives a user message with its session context, manages the tool-use loop with Claude, executes tools via MCP, and returns the final text to the FastAPI endpoint.

It is completely UI-agnostic: it does not know whether input comes from a browser, a test, or a CLI. It receives inputs and returns outputs.

---

## Location in the architecture

```bash id="x8v2mi"
FastAPI /chat
    │
    ▼
ChatEngine          ← this module
    ├── anthropic.Anthropic  → Claude
    └── McpStdioClient       → MCP Server
```

---

## Per-request instantiation

`ChatEngine` is instantiated once per request in FastAPI, not at server startup. This is a direct consequence of the stateless model (ADR 0001): the user’s API Key arrives in the `X-API-Key` header of each request and is used to construct the Anthropic client at that moment.

```python id="9qapqn"
# In FastAPI /chat endpoint
engine = ChatEngine(
    apiKey=request.headers["X-API-Key"],
    model=MODEL,
    mcpCmd=MCP_FEL_CMD,
    systemPrompt=SYSTEM_PROMPT,
    allowedRoots=[f"/tmp/fel/{session_uuid}"],
    routerDebug=ROUTER_DEBUG,
)
engine.start()
turn = engine.chatTurn(history, userText, xmlPath, pdfConfig)
engine.stop()
```

`allowedRoots` is built using the `session_uuid` from the JWT: each engine instance can only access its own session directory.

---

## Lifecycle

```bash id="qur5pr"
engine.start()
    └── Launches MCP server as subprocess
    └── Sends initialize → receives capabilities
    └── Calls tools/list → caches catalog

engine.chatTurn(...)
    └── Executes tool-use loop
    └── Returns finalText + metadata

engine.stop()
    └── Terminates MCP subprocess
```

`start()` and `stop()` are called within the same request. The MCP subprocess lives exactly as long as the message processing.

---

## `ChatEngine` class

### Attributes

| Attribute      | Type                  | Description                                    |
| -------------- | --------------------- | ---------------------------------------------- |
| `client`       | `anthropic.Anthropic` | Official Anthropic client                      |
| `model`        | `str`                 | Model to use (e.g. `claude-sonnet-4-20250514`) |
| `systemPrompt` | `str`                 | Base agent instruction                         |
| `routerDebug`  | `bool`                | Enables tool-use traces in response            |
| `allowedRoots` | `list[str]`           | Allowed sandbox paths                          |
| `mcp`          | `McpStdioClient`      | Active MCP client                              |
| `toolsCatalog` | `dict`                | Cached tool catalog from `start()`             |

---

## Helper functions

### `usageDict(resp) -> dict`

Extracts token usage metrics from Anthropic response:

```python id="t3y8fx"
{
    "input_tokens": int,
    "output_tokens": int,
    "cache_creation_input_tokens": int,
    "cache_read_input_tokens": int
}
```

Returns `{}` if `.usage` is not present.

---

### `serializeBlocks(blocks) -> list[dict]`

Converts SDK blocks (`TextBlock`, `ToolUseBlock`, `ToolResultBlock`) into JSON-serializable dicts for logging and tracing.

Fields by type:

| Type          | Serialized fields                |
| ------------- | -------------------------------- |
| `text`        | `type`, `text`                   |
| `tool_use`    | `type`, `id`, `name`, `input`    |
| `tool_result` | `type`, `tool_use_id`, `content` |

---

### `buildAnthropicTools(toolsCatalog) -> list[dict]`

Converts MCP tool catalog (`tools/list`) into Anthropic tool format:

```python id="d8q9st"
# Input (MCP format)
{
    "tools": [
        {
            "name": "fel_render",
            "description": "...",
            "inputSchema": { "type": "object", "properties": {...} }
        }
    ]
}

# Output (Anthropic format)
[
    {
        "name": "fel_render",
        "description": "...",
        "input_schema": { "type": "object", "properties": {...} }
    }
]
```

---

### `isPathAllowed(pathStr, allowedRoots) -> bool`

Verifies that a path is within allowed directories.

```python id="vjj3gr"
# Logic
resolved = Path(pathStr).resolve()
for root in allowedRoots:
    base = Path(root).resolve()
    if str(resolved).startswith(str(base) + os.sep):
        return True
return False
```

Protects against directory traversal (`../`) and cross-session access. A path like `/tmp/fel/other_session/` fails validation when `allowedRoots = ["/tmp/fel/current_session/"]`.

---

### `sanitizeMcpArgs(args, allowedRoots) -> dict`

Validates all path arguments before passing them to the MCP server.

Verified paths: `xml_path`, `logo_path`, `out_path`, `dir_xml`, `out_dir`.

Returns original dict if all paths are valid.

Raises `ValueError` if any path violates the sandbox.

---

### `contentBlocksToParams(blocks) -> list[dict]`

Converts SDK blocks into `ContentBlockParam` format for reinjection into Anthropic message history.

Filters only `text` and `tool_use` blocks. Discards the rest.

---

## `chatTurn` — main loop

### Signature

```python id="l40y5v"
def chatTurn(
    self,
    history: list[dict],
    userText: str,
    xmlPath: str | None = None,
    pdfConfig: PdfConfig | None = None,
    maxHops: int = 3,
) -> dict[str, Any]:
```

### Inputs

| Parameter   | Description                                            |
| ----------- | ------------------------------------------------------ |
| `history`   | Previous session messages (text only, no tool results) |
| `userText`  | Current user message                                   |
| `xmlPath`   | Absolute path to attached XML (within session sandbox) |
| `pdfConfig` | Visual configuration built from JWT                    |
| `maxHops`   | Max loop iterations (default 3)                        |

### Output

```python id="t1ovwz"
{
    "finalText": str,
    "router": {
        "trace": [...]
    },
    "tools": {
        "calls": [...]
    }
}
```

---

### Internal flow step-by-step

```bash id="ny4qdc"
1. Build messages[]
   └── history + { role: "user", content: userText }
   └── If xmlPath: inject file context into message

2. Prepare tools
   └── buildAnthropicTools(self.toolsCatalog)

3. Call Claude
   └── client.messages.create(
           model, tools, tool_choice="auto",
           system=systemPrompt, messages
       )

4. Split response blocks
   ├── toolUses  → tool_use blocks
   └── textBlocks → text blocks

5. If routerDebug: add snapshot to trace[]
   └── serializeBlocks(resp.content) + usageDict(resp)

6. Are there tool_use blocks?

   YES →
     a. Convert resp.content to assistantParams
     b. For each tool_use:
        i.  Sanitize args with sanitizeMcpArgs
        ii. Call self.mcp.callTool(name, safeArgs)
        iii.Parse result with parseTextBlock
        iv. Build tool_result block
        v.  Append to toolCalls[]
     c. Insert into messages[]:
        - assistantParams (model intent)
        - resultsForModel (tool outputs)
        - nudge: "Return one concise answer in the user's language;
                  do not show raw JSON."
     d. Continue to next hop

   NO →
     └── Concatenate textBlocks as finalText
     └── Return { finalText, router, tools }

7. If maxHops exceeded without final text:
   └── Return { finalText: "(no answer)", router, tools }
```

---

### XML context in message

When `xmlPath` is present, it is added explicitly to the user message before sending to Claude:

```bash id="32y4jw"
The user attached a FEL invoice.
File path: /tmp/fel/<session_uuid>/<file_uuid>.xml
Use this path when calling fel_validate or fel_render.
```

Claude receives the path and uses it in tool calls.

The model never reads XML content directly: it only orchestrates.

---

### Handling `ok=False` in validation

If `fel_validate` returns `ok=False`, the result is included in `tool_result` and Claude evaluates it. The model decides whether to:

* Continue with `fel_render` while informing the user of issues
* Stop and request user confirmation

This is Variant 1 defined in ADR 0002. Claude summarizes issues in natural language, without raw JSON.

---

## System prompt

The system prompt instructs the agent to:

* Always respond in the user’s language
* Use MCP tools when required
* Never return raw JSON: always natural language
* On validation: summarize totals and issues clearly
* On rendering: clearly indicate generated PDF name or path
* On batch: report how many files were processed

Configurable via `SYSTEM_PROMPT` in `.env`. If not set, a default from `settings.py` is used.

---

## Logging and tracing

When `ROUTER_DEBUG=True`:

* Each loop hop adds a snapshot to `router.trace[]`
* Each executed tool call is added to `tools.calls[]`
* Full output is available in `/chat` response
* FastAPI can expose it to the frontend for optional logs view

When `ROUTER_DEBUG=False`:

* `router` and `tools` are empty
* No serialization overhead

---

## Session path sandbox

The sandbox is the most critical protection in ChatEngine (web mode).

Each instance receives `allowedRoots` built from the request’s JWT `session_uuid`:

```python id="mpwt37"
allowedRoots = [f"/tmp/fel/{session_uuid}"]
```

This ensures that even with concurrent users, one user’s tool calls cannot access another’s files. Any attempt to access paths outside the sandbox raises `ValueError` before reaching the MCP server.

---

## Differences from previous code

| Aspect        | Previous code           | This version            |
| ------------- | ----------------------- | ----------------------- |
| Instantiation | Once at CLI startup     | Once per HTTP request   |
| API Key       | From server `.env`      | From `X-API-Key` header |
| allowedRoots  | Global, static          | Per session (from JWT)  |
| xmlPath       | Manual CLI argument     | From session context    |
| pdfConfig     | From `.env` / config.py | From JWT payload        |
| Frontend      | CLI (rich + REPL)       | FastAPI endpoint        |

---

## Related files

* `adr/0002-mcp-as-orchestration-layer.md` → MCP architecture decision
* `adr/0001-stateless-session-model.md` → why per-request instantiation
* `adr/0003-user-owned-api-key.md` → why API key is header-based
* `technical/mcp-server.md` → tools the engine can invoke
* `technical/session-model.md` → how xmlPath and pdfConfig reach engine
* `technical/pdf-generation.md` → PdfConfig structure
