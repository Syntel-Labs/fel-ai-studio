# Session Model — FEL AI Studio

## Principle

The server does not store user state permanently. Everything needed to process a request is carried in the JWT or arrives within the request itself.

The session exists only in the browser and for the lifetime of the JWT.

Temporary files (XML, PDF) are the only exception: they exist on disk for the duration of the JWT and are automatically deleted upon expiration.

---

## What a session is

A session represents a user’s working intent: it has a defined visual configuration and may include an attached XML to process.

It does not represent an identity, has no persistent history, and does not survive tab closure.

---

## What lives where

**In the browser (sessionStorage) — authoritative:**

* User API Key (never sent to the server except per request header)
* Full visual configuration (7 color tokens, logo in base64)
* User-defined PDF name
* Reference to the active token

**In the JWT:**

* 7 color tokens (`primary`, `primary_text`, `secondary`, `secondary_text`, `text_main`, `text_soft`, `background`)
* Logo reference (`logo_ref`: UUID of uploaded file)
* PDF name template (`pdf_name`: e.g. `factura_{serie}_{dte}.pdf`)
* Session directory path `/tmp/fel/<session_uuid>/`
* `iat` and `exp`

**In `/tmp/fel/<session_uuid>/` (temporary):**

* Uploaded XML: `<file_uuid>.xml`
* Generated PDF (ephemeral): `<file_uuid>.pdf` — deleted after download

---

## XML isolation between users

Each session has its own UUID v4 subdirectory. Two users uploading files simultaneously never share paths nor can access each other’s files.

```bash id="6lh3gi"
/tmp/fel/
├── a3f9b1c2-..../          ← session user A
│   ├── xml_7d2e....xml
│   └── out_7d2e....pdf
├── b8e4d0f1-..../          ← session user B
│   └── xml_c1a3....xml
└── ...
```

The MCP server receives full paths as arguments and validates them against `ALLOWED_ROOTS` before processing. A path from another session fails sandbox validation.

---

## Session creation flow

```bash id="l5th9k"
1. User configures in panel:
   └── 7 color tokens, logo, PDF name

2. Browser sends POST /session:
   └── { config_visual }    ← no API Key, no XML

3. FastAPI:
   ├── Generates session_uuid (UUID v4)
   ├── Creates /tmp/fel/<session_uuid>/
   ├── Builds JWT payload with config_visual + session_uuid
   ├── Signs with SECRET_KEY (HS256)
   └── Returns { token, expires_at }

4. Browser stores in sessionStorage:
   └── { token, api_key, config_visual }

5. Generated link:
   └── https://app/chat?session=<jwt>
```

---

## XML upload flow

```bash id="42t7b6"
1. User attaches .xml file in chat UI

2. Browser sends POST /upload:
   └── multipart/form-data: { file: factura.xml }
   └── Header: Authorization: Bearer <jwt>

3. FastAPI middleware:
   ├── Validates JWT → extracts session_uuid
   ├── Verifies size (MAX_XML_SIZE_KB)
   ├── Discards original name → assigns file_uuid
   └── Saves to /tmp/fel/<session_uuid>/<file_uuid>.xml

4. Returns: { xml_ref: "<file_uuid>" }

5. Browser includes xml_ref in chat context
   └── Does not resend XML in each message
```

---

## Chat flow with attached XML

```bash id="u8kjiy"
1. User types: "generate the PDF of this invoice"

2. Browser sends POST /chat:
   └── { message, xml_ref: "<file_uuid>" }
   └── Header: Authorization: Bearer <jwt>
   └── Header: X-API-Key: <user_api_key>

3. FastAPI:
   ├── Validates JWT → extracts session_uuid + config_visual
   ├── Builds xml_path: /tmp/fel/<session_uuid>/<file_uuid>.xml
   ├── Builds PdfConfig from JWT config_visual
   └── Instantiates ChatEngine with API key

4. ChatEngine → Claude (tool_choice="auto")
   └── Claude emits tool_use: fel_render
       └── args: { xml_path, out_path, config: PdfConfig }

5. MCP Server generates PDF
   └── Saved at /tmp/fel/<session_uuid>/out_<file_uuid>.pdf

6. ChatEngine returns finalText + pdf_ref

7. FastAPI returns { response, pdf_ref }
   └── Frontend shows download button

8. GET /render?ref=<pdf_ref>
   └── StreamingResponse of PDF
   └── File deleted after sending
```

---

## Temporary file lifecycle

```bash id="m6o0hx"
POST /upload    → XML created  in /tmp/fel/<session>/<uuid>.xml
POST /chat      → PDF created  in /tmp/fel/<session>/out_<uuid>.pdf
GET  /render    → PDF sent → PDF deleted immediately

On JWT expiration:
  └── Background task detects expired session directory
  └── Deletes entire /tmp/fel/<session_uuid>/
```

Cleanup runs as a FastAPI background task at the start of each request, checking directories whose TTL has expired.

---

## JWT structure

```json id="92ub5o"
{
  "header": { "alg": "HS256", "typ": "JWT" },
  "payload": {
    "session_uuid": "a3f9b1c2-....",
    "colors": {
      "primary":        "#fe5101",
      "primary_text":   "#ffffff",
      "secondary":      "#343534",
      "secondary_text": "#ffffff",
      "text_main":      "#000000",
      "text_soft":      "#8c8d8e",
      "background":     "#ffffff"
    },
    "logo_ref":   "logo_<uuid>.jpg",
    "pdf_name":   "factura_{serie}_{dte}.pdf",
    "iat": 1710000000,
    "exp": 1710003600
  }
}
```

What never appears in the JWT:

* User API Key
* XML content
* Plaintext fiscal data

---

## Token validation per request

```bash id="v6gyc6"
Request arrives with:
  Authorization: Bearer <jwt>

FastAPI middleware:
  1. Decodes with SECRET_KEY
  2. Verifies exp
  3. Verifies /tmp/fel/<session_uuid>/ exists
  4. Injects payload into request context

If any step fails:
  └── 401 → frontend redirects to /config with visual config intact
```

---

## What happens on expiration

```bash id="c1yq3c"
Token expired → 401
  └── Frontend detects 401
        └── Banner: "Your session expired, reconfigure"
              └── Redirects to /config
                    └── Visual config restored from sessionStorage
                          └── User only re-uploads XML
```

---

## Expiration and cleanup

| Parameter          | Default               | Environment variable  |
| ------------------ | --------------------- | --------------------- |
| JWT duration       | 60 minutes            | `SESSION_TTL_MINUTES` |
| /tmp directory TTL | = JWT duration        | automatic             |
| /tmp cleanup       | Start of each request | not configurable      |
| PDF post-download  | Immediate             | not configurable      |

---

## Related files

* `adr/0001-stateless-session-model.md` → decision to avoid DB
* `adr/0005-ephemeral-link-generation.md` → JWT construction
* `adr/0003-user-owned-api-key.md` → why API Key is not in token
* `technical/pdf-generation.md` → PdfConfig and color tokens
* `technical/api-reference.md` → schemas for `/session`, `/upload`, `/chat`
