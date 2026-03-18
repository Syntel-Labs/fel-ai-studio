# Flow Diagrams — FEL AI Studio

Mermaid diagrams of the complete system flow in its two operating modes. Each diagram covers a different aspect of the system.

---

## 1. Full flow — Web mode (user session)

```mermaid
sequenceDiagram
    actor U as User
    participant B as Browser (React)
    participant F as FastAPI
    participant E as ChatEngine
    participant M as MCP Server
    participant R as ReportLab

    U->>B: Configures colors, logo, API Key
    Note over B: Everything in sessionStorage<br/>API Key never leaves the browser

    U->>B: Uploads XML + requests link
    B->>F: POST /session { config_visual }
    Note over F: Generates session_uuid<br/>Creates /tmp/fel/<session_uuid>/
    F-->>B: { token (JWT), expires_at }
    Note over B: Stores token in sessionStorage<br/>Builds link ?session=<jwt>

    U->>B: Opens chat with active link
    U->>B: Attaches factura.xml
    B->>F: POST /upload { file.xml }<br/>Authorization: Bearer <jwt>
    Note over F: Validates JWT<br/>Discards original name<br/>Stores as <file_uuid>.xml
    F-->>B: { xml_ref: "<file_uuid>" }

    U->>B: Types "generate the PDF of this invoice"
    B->>F: POST /chat { message, xml_ref }<br/>Authorization: Bearer <jwt><br/>X-API-Key: <key>
    Note over F: Validates JWT → extracts session_uuid<br/>Builds xml_path + PdfConfig<br/>Instantiates ChatEngine with api_key

    F->>E: chatTurn(history, message, xml_path, config)
    E->>E: buildAnthropicTools(catalog)
    E->>F: Calls Claude API with tool_choice="auto"
    Note over E: Claude emits tool_use: fel_validate

    E->>M: callTool("fel_validate", { xml_path })
    M-->>E: { ok, issues, totals }
    Note over E: Claude evaluates result<br/>Decides to continue with render

    E->>M: callTool("fel_render", { xml_path, out_path, config })
    M->>R: generatePdf(xmlPath, outputPdf, PdfConfig)
    R-->>M: PDF written to /tmp/fel/<session_uuid>/out_<uuid>.pdf
    M-->>E: { ok: true, pdf_path }

    E-->>F: { finalText, pdf_ref }
    F-->>B: { response, pdf_ref }
    B-->>U: Displays response + download button

    U->>B: Downloads PDF
    B->>F: GET /render?ref=<pdf_ref><br/>Authorization: Bearer <jwt>
    F-->>B: StreamingResponse (PDF)<br/>Content-Disposition: attachment
    Note over F: File deleted after sending
    B-->>U: PDF downloaded
```

---

## 2. Security flow — Validation layers per request

```mermaid
flowchart TD
    A([Incoming request]) --> B{PREVIEW_TOKEN\nvalid?}
    B -->|No| B1([401 Unauthorized])
    B -->|Yes| C{HTTPS in\nproduction?}
    C -->|No — ENV=production| C1([Reject HTTP])
    C -->|Yes| D{Origin in\nCORS_ORIGINS?}
    D -->|No| D1([403 Forbidden])
    D -->|Yes| E{Rate limit\nper IP OK?}
    E -->|Limit exceeded| E1([429 Too Many Requests\nRetry-After header])
    E -->|OK| F{Valid and\nsanitized input?}
    F -->|No| F1([422 Unprocessable Entity])
    F -->|Yes| G{JWT valid\nand not expired?}
    G -->|No| G1([401 — redirect to /config\nwith visual config intact])
    G -->|Yes| H([Normal processing])
```

---

## 3. Session flow — Full lifecycle

```mermaid
stateDiagram-v2
    [*] --> No_session : User opens the app

    No_session --> Configuring : Enters API Key\nand configures colors/logo

    Configuring --> Active_session : POST /session success\nJWT generated

    Active_session --> XML_uploaded : POST /upload success\nXML in /tmp/fel/<session_uuid>/

    XML_uploaded --> Generating : User requests PDF\nChatEngine running

    Generating --> PDF_ready : fel_render success\nPDF in /tmp/

    PDF_ready --> XML_uploaded : User downloads PDF\nPDF deleted after sending\nSession remains active

    Active_session --> Expired : JWT TTL expired\n(default 60 min)
    XML_uploaded --> Expired : JWT TTL expired
    Generating --> Expired : JWT TTL expired

    Expired --> Configuring : Frontend detects 401\nVisual config recovered\nfrom sessionStorage\nOnly re-upload XML

    Active_session --> [*] : User closes tab\nsessionStorage cleared
    Expired --> [*] : User leaves
```

---

## 4. MCP flow — Internal tool-use loop

```mermaid
sequenceDiagram
    participant F as FastAPI /chat
    participant E as ChatEngine
    participant C as Claude API
    participant M as MCP Server
    participant R as ReportLab

    F->>E: chatTurn(history, userText, xml_path, config)
    Note over E: Builds messages[]<br/>Prepares tools catalog

    loop Until maxHops=3 or no tool_use
        E->>C: messages.create(tools, tool_choice="auto")

        alt Claude emits tool_use
            C-->>E: [tool_use: fel_validate | fel_render | fel_batch]
            Note over E: Sanitizes args<br/>Verifies sandbox path

            E->>M: JSON-RPC tools/call { name, arguments }

            alt fel_validate
                M-->>E: { ok, issues, totals }
            else fel_render
                M->>R: generatePdf(xmlPath, outputPdf, PdfConfig)
                R-->>M: PDF written
                M-->>E: { ok: true, pdf_path }
            else fel_batch
                M->>R: generatePdf() × N files
                R-->>M: N PDFs + manifest.json
                M-->>E: { ok, count, manifest_path }
            end

            E->>C: Reinserts tool_result into messages[]
            Note over E: "Return concise answer\nin user's language"

        else Claude emits only text
            C-->>E: [text: final response]
            E-->>F: { finalText, router.trace, tools.calls }
        end
    end

    Note over E: If maxHops exceeded → "(no answer)"
```

---

## 5. Docker Hub mode flow — Claude Desktop

```mermaid
sequenceDiagram
    actor U as User
    participant CD as Claude Desktop
    participant D as Docker Container\n(felai/mcp-server)
    participant CF as fel_config.json\n(local volume)
    participant R as ReportLab

    U->>CD: Configures claude_desktop_config.json\nwith MCP_TOKEN + volume
    U->>CF: Creates /data/fel_config.json\nwith colors, logo, footer

    CD->>D: docker run --rm -i\n-e MCP_TOKEN=<secret>\n-v /data:/app/data

    D->>D: Verifies MCP_TOKEN
    D->>CF: Reads fel_config.json → PdfConfig
    Note over D: Falls back to defaults\nif file does not exist

    CD->>D: initialize {}
    D-->>CD: { protocolVersion, capabilities }

    CD->>D: tools/list
    D-->>CD: [fel_validate, fel_render, fel_batch]

    U->>CD: Attaches factura.xml\n"Generate the PDF with my colors"
    CD->>D: tools/call fel_validate\n{ xml_path: "/app/data/xml/factura.xml" }
    D-->>CD: { ok, issues, totals }

    CD->>D: tools/call fel_render\n{ xml_path, out_path, config? }
    Note over D: Hierarchy: args > fel_config.json\n> env vars > defaults
    D->>R: generatePdf(xmlPath, outputPdf, PdfConfig)
    R-->>D: PDF written in /app/data/out/
    D-->>CD: { ok: true, pdf_path }

    CD-->>U: "PDF generated in /data/out/factura_9E52C854.pdf"
    Note over U: File available\nin local folder
```

---

## 6. Temporary file lifecycle — Web mode

```mermaid
flowchart LR
    A([POST /session]) -->|Creates| B[/tmp/fel/\nsession_uuid/]
    C([POST /upload]) -->|Writes| D[xml_uuid.xml]
    D --> B
    E([POST /chat → fel_render]) -->|Writes| F[out_uuid.pdf]
    F --> B
    G([GET /render]) -->|Reads and deletes| F
    H([JWT expires\nBackground task]) -->|Deletes entire directory| B
    B -->|Contains| D
    B -->|Contains| F

    style B fill:#343534,color:#fff
    style D fill:#fe5101,color:#fff
    style F fill:#fe5101,color:#fff
```

---

## Related files

* `architecture/overview.md` → prose description of each component
* `architecture/stack.md` → justification of each technology
* `technical/session-model.md` → details of JWT and session isolation
* `technical/engine.md` → ChatEngine and tool-use loop implementation
* `technical/mcp-server.md` → JSON-RPC protocol and exposed tools
