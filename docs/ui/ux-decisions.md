# UX Decisions — FEL AI Studio

## Purpose

This document records the user experience decisions that define how the interface behaves and feels. It is not a visual design spec — it captures the reasoning behind each flow, structure, and behavior decision that affects the user.

---

## UX principles of the project

Before any specific decision, three principles guide the interface:

**1. The user should not need to understand how the system works**
Rodrigo (SMB) does not know what MCP or JWT are. The interface must fully abstract technical complexity.

**2. Visual configuration is the tangible product**
The generated link is what the user takes away. The entire flow converges there. The UI should make it feel like an outcome, not a step.

**3. No friction in the critical path**
The critical path is: configure → upload XML → generate PDF. Anything that does not contribute to this path is noise.

---

## Application structure

### Decision: sidebar with three fixed sections

```bash
┌──────────┬─────────────────────────────┐
│          │                             │
│  HOME    │                             │
│          │      Main area              │
│  CONFIG  │                             │
│          │                             │
│  CHAT    │                             │
│          │                             │
└──────────┴─────────────────────────────┘
```

**Why sidebar instead of tabs or navbar:**
Users need to switch sections without losing context. A top navbar reduces workspace. Tabs imply interchangeability — these sections are hierarchical (HOME explains, CONFIG configures, CHAT produces).

**Why only three sections:**
Each has a single responsibility. Splitting further fragments the flow without improving the critical path.

---

## HOME section

### Decision: interactive README, not a landing page

HOME is not marketing. It is a functional explanation: what it does, how it works, what the user needs.

**Content:**

* What FEL AI Studio is (one sentence)
* What the user needs (API Key + XML)
* 4-step visual flow
* Simplified architecture diagram
* Direct link to CONFIG

**Why not onboarding wizard:**
No persistent sessions exist. A wizard implies saved progress. HOME is a reference, not a forced entry.

---

## CONFIG section

### Decision: single-page configuration (no steps)

All configuration fields are on one page.

**Why:**
Fields are independent. Sequential steps would imply dependencies that do not exist.

---

### Decision: real-time PDF preview

The panel shows a live preview that updates instantly.

```bash
┌─────────────────┬──────────────────────┐
│  Configuration  │     PDF Preview      │
│                 │                      │
│  Colors         │  ┌────────────────┐  │
│  Logo           │  │  [top bar]     │  │
│  Footer         │  │  LOGO | data   │  │
│  PDF Name       │  │  items table   │  │
│                 │  │  totals       │  │
│  [Generate      │  │  [footer]     │  │
│   link]         │  └────────────────┘  │
└─────────────────┴──────────────────────┘
```

**Why:**
Color tokens are abstract. Immediate visual feedback is the only way non-technical users understand them.

**Implementation:**
Pure React component using mock data. No backend calls.

**Why mock data:**
XML belongs to CHAT, not CONFIG. Preview must work independently.

---

### Decision: disable “Generate link” without API Key

**Why client-side blocking:**
Failing early prevents frustration and educates the user.

**Message:**

> "You need to configure your Anthropic API Key to generate the link."

---

### Decision: API Key validated locally only

**Why not validate via API:**
Avoid unnecessary exposure and latency. Format is sufficient.

Errors surface naturally during chat if invalid.

---

### Decision: semantic color roles

**Why:**
“primary” is meaningless to Rodrigo. “Top bar color” is clear.

---

### Decision: visible expiration

**Why:**
Users must understand time constraints when sharing links.

---

## CHAT section

### Decision: standard chat UI with attachment

**Why attach in chat:**
XML is per-conversation input, not persistent config.

---

### Decision: inline download button

**Why:**
The PDF is the result of that message. Navigation breaks flow.

---

### Decision: optional process + logs panels

**Why optional:**
Rodrigo does not need them. Diego does.

**Why tabs:**
Preserve screen space.

---

### Decision: ephemeral chat history

**Why:**
Consistent with stateless model (ADR 0001).

**On expiration:**

> "Your session expired. Your visual configuration is saved. Go to CONFIG to generate a new link."

---

## Global behaviors

### Decision: sessionStorage, not localStorage

**Why:**
Security + aligns with ephemeral model.

---

### Decision: no confirmation modals

**Why:**
All actions are reversible. Modals add friction without value.

---

### Decision: inline errors, not toasts

**Why:**
Errors must persist until resolved.

**Exception:**
Global issues → top banner.

---

### Decision: simple mode by default

**Why:**
Primary user is non-technical. Advanced views are opt-in.

---

## Resolved decisions

* PDF preview: React-rendered (no backend)
* Invalid XML: rejected at `/upload` with inline error
* Tabs state persists per session (React state only)

---

## Related files

* `ui/config-panel.md` → configuration panel details
* `product/personas.md` → user profiles driving decisions
* `architecture/overview.md` → underlying system flow
* `adr/0001-stateless-session-model.md` → no persistence
* `adr/0003-user-owned-api-key.md` → API Key handling
