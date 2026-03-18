# ADR 0003 — User-Owned API Key

**Status:** Accepted
**Date:** 2025-03-17

---

## Context

FEL AI Studio uses Claude (Anthropic) as the main agent to interpret requests and orchestrate MCP tools. Each call to the Anthropic API has a direct cost associated with token usage.

In the previous code, the API Key lived in the server’s `.env`, which means any user of the system would consume from the same operator budget. For an academic project this is acceptable. For a publicly accessible portfolio or a real product, it is not.

This decision applies to web mode. Docker Hub mode (ADR 0006) has its own API Key model: the user configures it as an environment variable in the container.

Three management models were evaluated:

---

## Options considered

### Option A — Operator API Key (previous model)

The key lives on the server. The operator absorbs all costs.

**Advantage:** simpler user experience, no initial friction.

**Disadvantage:** uncontrolled operational cost, risk of abuse, the operator assumes financial responsibility for every request.

**Verdict:** discarded. Not viable for public access.

---

### Option B — User API Key, stored on the server

The user enters their key, the server stores it encrypted per session.

**Advantage:** the user controls their consumption, the server can validate the key.

**Disadvantage:** the server handles and stores a user secret, even if temporarily. Requires encryption, secure management, and creates legal responsibility over third-party data.

**Verdict:** discarded. Contradicts the system’s privacy principle.

---

### Option C — User API Key, only in the browser

The user enters their key in the configuration panel. It is stored in `sessionStorage`. It travels to the server only in the `X-API-Key` header of each request and is used at that moment to instantiate the Anthropic client. It is not persisted, not logged, and not included in the JWT.

**Advantage:** the server never stores the secret. The user has full control. Zero operational cost for the system. Aligned with the stateless model.

**Disadvantage:** the user needs an active Anthropic account and must know where to obtain their API Key. Adds friction during onboarding.

**Verdict:** accepted.

---

## Decision

**The user’s API Key travels in the `X-API-Key` header of each request and is used exclusively to instantiate the `ChatEngine` in the context of that request. It is not persisted in any layer of the system.**

---

## Consequences

**Positive:**

* Zero operational cost for the system operator
* The server is never responsible for third-party secrets
* The user can revoke their key from Anthropic without affecting the system
* Aligned with the privacy principle and stateless design

**Negative:**

* The user needs an active Anthropic account
* Adds friction on first use
* If the user does not have a key, the system does not work at all
* The `X-API-Key` header must travel over HTTPS mandatorily

**Neutral:**

* The configuration panel must include clear instructions on how to obtain an Anthropic API Key
* The frontend validates key format before allowing link generation

---

## Constraints imposed by this decision

* `POST /session` does not accept requests without `X-API-Key` present
* The session JWT never includes the API Key
* Server logs never record the value of the `X-API-Key` header
* The frontend blocks link generation if sessionStorage does not have a configured key

---

## Future review

If the project evolves into a SaaS model with paid plans, this decision will be superseded by a new ADR defining key management per user account with encryption in a database.

---

## Related files

* `adr/0001-stateless-session-model.md` → context of the stateless model
* `adr/0006-docker-hub-mcp-distribution.md` → key model in Docker mode
* `technical/session-model.md` → how the key travels and what the JWT does not contain
* `technical/engine.md` → how ChatEngine is instantiated per request
