# ADR 0005 — Ephemeral Link Generation

**Status:** Accepted
**Date:** 2025-03-17

---

## Context

The core flow of FEL AI Studio culminates in a link that the user can open, share, or save to start a conversation with the AI agent about their invoice. This link must:

* Contain enough context for the server to process the invoice
* Not expose sensitive data in the URL
* Have a limited lifetime
* Not require the server to remember anything between generation and use

The link is the tangible product the user gets from the configuration panel. The entire app flow converges at this point.

---

## Options considered

### Option A — URL with plaintext parameters

```bash
https://app/chat?nit=12345&colors=%23fe5101&xml=...
```

**Verdict:** discarded. Fiscal data visible in URL, logs, and Referer.

---

### Option B — Opaque ID with server-side state

```bash
https://app/chat?session=a3f9b1c2
```

**Verdict:** discarded. Requires server-side state, contradicts ADR 0001.

---

### Option C — JWT as session parameter

```bash
https://app/chat?session=eyJhbGc...
```

The JWT contains the necessary context, is signed, and has expiration.

The server stores nothing: it verifies the signature and extracts the payload.

**Advantage:** stateless, self-contained, native expiration, verifiable without a database.

**Disadvantage:** longer URL than an opaque ID.

**Verdict:** accepted.

---

## Decision

**The session link is a URL with a signed JWT as the query parameter `?session=<jwt>`. The server verifies the signature and extracts context on each request. There is no server-side state associated with the link.**

---

## JWT Payload — Option A (accepted)

The JWT contains the full visual configuration and the `session_uuid` that identifies the user’s temporary directory on the server.

```json
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
    "logo_ref":  "logo_<uuid>.jpg",
    "pdf_name":  "factura_{serie}_{dte}.pdf",
    "iat": 1710000000,
    "exp": 1710003600
  }
}
```

The `session_uuid` references `/tmp/fel/<session_uuid>/` on the server.

The directory exists while the JWT is valid and is deleted upon expiration.

**Reason for choosing over other options:**

* Option B (visual config only): XML must be sent on every request, increasing payload and complicating the frontend
* Option C (XML in base64): token becomes too large to share, real invoices can exceed several KB

---

## Link generation

```bash
POST /session
  Body: { config_visual }
  Header: X-API-Key

  └── FastAPI:
        1. Generates session_uuid (UUID v4)
        2. Creates /tmp/fel/<session_uuid>/
        3. Builds JWT payload with config_visual + session_uuid
        4. Signs with SECRET_KEY (HS256)
        5. Returns { token, expires_at }

Browser:
  └── https://app/chat?session=<token>
  └── Stores token in sessionStorage
```

---

## Link usage

```bash
User opens https://app/chat?session=<jwt>

Frontend:
  1. Extracts token from query param
  2. Stores in sessionStorage if not present
  3. Attaches in every request: Authorization: Bearer <jwt>

FastAPI middleware:
  1. Decodes and verifies signature
  2. Verifies exp
  3. Verifies that /tmp/fel/<session_uuid>/ exists
  4. Injects payload into request context
```

---

## What never goes in the JWT

* User API Key
* Passwords or credentials of any kind
* XML content
* Plaintext fiscal data
* Absolute filesystem paths of the host

---

## Defined parameters

| Parameter   | Value                                 | Configurable               |
| ----------- | ------------------------------------- | -------------------------- |
| Algorithm   | HS256                                 | No                         |
| Secret      | `SECRET_KEY` in .env                  | No (required in deploy)    |
| Default TTL | 60 minutes                            | Yes, `SESSION_TTL_MINUTES` |
| Transport   | `?session=` + `Authorization: Bearer` | No                         |

---

## Consequences

**Positive:**

* Self-contained link verifiable without a database
* Native expiration via JWT standard
* Scales horizontally without coordination
* An expired link is useless even if intercepted

**Negative:**

* Longer URL than an opaque ID
* If `SECRET_KEY` rotates, all active links are invalidated
* The user cannot extend a link’s lifetime without regenerating it
* The server must actively manage `/tmp/` cleanup

**Neutral:**

* The panel shows the expiration date of the generated link
* The frontend handles expired links by redirecting to `/config` with visual config preserved from sessionStorage

---

## Future review

If multi-tenant or user account support is added, links could be associated with an identity with automatic renewal. This requires superseding both this ADR and ADR 0001.

---

## Related files

* `adr/0001-stateless-session-model.md` → principle that motivates this design
* `adr/0003-user-owned-api-key.md` → what is excluded from the link
* `technical/session-model.md` → session model implementation
* `technical/api-reference.md` → full schema of POST /session
