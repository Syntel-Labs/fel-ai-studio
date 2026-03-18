# ADR 0001 — Stateless Session Model

**Status:** Accepted

**Date:** 2025-03-17

---

## Context

FEL AI Studio processes real fiscal invoices (FEL XMLs issued by the SAT of Guatemala). These documents contain sensitive data: issuer NIT, recipient NIT or CUI, amounts, dates, and fiscal authorization number.

The earliest and most fundamental decision of the system is: does the server store any of this?

In the previous code there was no database, but there was also no defined session model. The state lived in the CLI process while it was running. When moving to a web architecture with multiple concurrent users, this ambiguity becomes a design problem that must be resolved explicitly.

Three models were evaluated:

---

## Options considered

### Option A — Persistent session with database

The server stores user configuration, processed invoice history, and visual preferences in a database (PostgreSQL, SQLite, etc.).

**Advantage:** richer experience, the user can resume previous sessions, view history, and recover generated PDFs.

**Disadvantage:** the server becomes the custodian of sensitive fiscal data. It requires encryption at rest, a retention policy, and compliance with privacy regulations. It drastically increases deployment complexity and the legal responsibility of the operator.

**Verdict:** discarded. It contradicts the core value proposition of the system.

---

### Option B — In-memory server session (Redis / in-process)

The server stores session state in memory with TTL, without disk persistence.

**Advantage:** simpler than a DB, the state disappears automatically.

**Disadvantage:** the server is still a temporary custodian of sensitive data. It adds an infrastructure dependency (Redis) or limits the system to a single process. It does not scale horizontally without additional coordination.

**Verdict:** discarded. Simplicity does not outweigh the risk.

---

### Option C — Stateless session with JWT

The server does not store any user state. Everything needed to process a request travels in the signed JWT or in the request itself. The server is completely memoryless between requests.

**Advantage:** the server is never a custodian of user data. Simple deploy, scales horizontally without coordination. Aligned with the privacy promise. Reduces attack surface.

**Disadvantage:** state that does not fit in the JWT must go in the request or in ephemeral temporary storage with active management. The server cannot offer history or recovery of previous sessions.

**Verdict:** accepted.

---

## Decision

**The server does not store user state in any persistent layer. Each request is self-contained. Session state lives in the browser (sessionStorage) and in a signed JWT with TTL. Temporary files (XML, PDF) are the only exception: they exist on disk for the duration of the JWT and are automatically deleted upon expiration.**

---

## What “stateless” means in this system

| Layer                | State      | Where it lives                                   |
| -------------------- | ---------- | ------------------------------------------------ |
| Visual configuration | In session | Browser sessionStorage                           |
| API Key              | In session | Browser sessionStorage                           |
| Session token        | In session | sessionStorage + query param                     |
| Invoice XML          | Ephemeral  | /tmp/fel/<session_uuid>/ (TTL = JWT)             |
| Generated PDF        | Ephemeral  | /tmp/fel/<session_uuid>/ (deleted post-download) |
| Chat history         | In session | React component memory                           |
| RPC logs             | Ephemeral  | Local file, not indexed by user                  |

None of the above survives tab closure or JWT expiration.

---

## Consequences

**Positive:**

* The server is not responsible for user fiscal data
* Deployment without database or distributed cache dependencies
* Reduced attack surface: there is no data to steal from the server
* The privacy promise is verifiable by inspecting the code

**Negative:**

* No history: the user cannot recover a previous session
* If the JWT expires during work, the user must restart the session
* The XML must be re-uploaded in each new session
* The chat has no memory between sessions

**Neutral:**

* The frontend must manage session state explicitly
* Temporary file cleanup is the server’s responsibility
* System logs must not contain identifiable user data

---

## Constraints imposed by this decision

* A user database cannot be added without superseding this ADR
* Invoice history cannot be stored without superseding this ADR
* Files in `/tmp/fel/` must have TTL equal to the JWT and be actively cleaned via a background task in FastAPI
* The frontend cannot assume the server remembers anything between requests
* Each session has its own UUID subdirectory: two users never share paths nor can access each other’s files

---

## Future review

If the project evolves into a SaaS with user accounts, this decision will be superseded by a new ADR defining the persistence model, encryption at rest, and data retention policy. In that scenario, this ADR remains as a reference of the original privacy model.

---

## Related files

* `adr/0003-user-owned-api-key.md` → direct consequence of this model
* `adr/0005-ephemeral-link-generation.md` → how the JWT is constructed
* `technical/session-model.md` → detailed implementation of the model
* `product/vision.md` → privacy promise as a value proposition
