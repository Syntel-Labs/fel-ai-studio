# Personas — FEL AI Studio

## Context

This document defines the three main profiles that interact with FEL AI Studio. It serves to guide UX decisions, prioritize features, and communicate the product’s value according to the audience.

---

## Persona 1 — The potential customer

**Name:** Rodrigo Estrada

**Role:** Owner of a Guatemalan SMB (distribution, consulting, retail)

**Context:** Regularly issues FEL invoices through the SAT portal or their accounting system. The invoices delivered to clients look generic and lack brand identity. Has no development team.

### What he needs

* Upload his invoice XML and obtain a PDF with his logo and colors
* A fast process, no registration, no installation
* Assurance that no one permanently stores his fiscal data
* Control over how each section of the invoice looks (colors, footer, table)

### What he does not tolerate

* Long forms or onboarding processes
* Being asked for an account or password
* Technical interfaces or developer jargon

### How he finds the product

Referred by another business owner, social media ads, or direct search.

He lands on the page, configures in 2 minutes, and generates his first link.

### Success metric for this persona

Time from opening the app to downloading the first PDF < 3 minutes.

---

## Persona 2 — The recruiter or tech company

**Name:** Valeria Méndez

**Role:** Tech recruiter or engineering manager at a software company

**Context:** Reviews GitHub profiles looking for evidence of real work, not just tutorials. Interested in how someone designs systems, makes technical decisions, and documents their work.

### What she needs

* Understand the project in under 5 minutes without cloning the repo
* See documented architectural decisions (ADRs)
* Confirm the stack is modern and relevant to her company
* Evidence that the developer can integrate AI into real systems

### What she does not tolerate

* Generic READMEs without context
* Projects without clear installation instructions
* Code without structure or documentation

### How she finds the product

GitHub portfolio, LinkedIn, or direct candidate reference.

She reads the README, reviews `docs/`, clones and runs `docker-compose up`.

### Success metric for this persona

The project runs locally with a single command and the architecture is understandable without talking to the author.

---

## Persona 3 — The technical community

**Name:** Diego Fuentes

**Role:** Backend or full-stack developer interested in applied AI

**Context:** Follows the LLM and MCP ecosystem. Looks for real implementation references, not just official Anthropic examples. Interested in how Claude integrates with existing systems without high-level abstractions.

### What he needs

* Clean, well-structured code as a technical reference
* Understand how MCP is implemented with Claude in a real use case
* Honest technical documentation including limitations and design decisions
* Inspiration for his own projects, not copy-paste code

### What he does not tolerate

* Overengineered code or unnecessary dependencies
* Documentation that hides trade-offs
* Projects that only work on the author’s machine

### How he finds the product

GitHub search ("MCP server Python", "Claude tool use FastAPI"), technical posts, or Latin American developer communities.

### Success metric for this persona

He understands the architecture and technical decisions just by reading `docs/` and the README, without cloning the repo.

---

## Comparative summary

|                      | Rodrigo (SMB)     | Valeria (Recruiter)    | Diego (Dev)               |
| -------------------- | ----------------- | ---------------------- | ------------------------- |
| Entry point          | Landing / UI      | GitHub / README        | GitHub / docs             |
| Expected depth       | UI only           | README + docs/         | README + docs/            |
| Decision made        | Use the product   | Contact the dev        | Learn and get inspired    |
| What they value most | Speed and privacy | Architecture and stack | Technical clarity         |
| Drop-off risk        | Long process      | Incomplete docs        | Superficial documentation |

---

## License and project usage

FEL AI Studio is a personal portfolio project. The code and documentation are public for technical demonstration and learning purposes.

Not allowed:

* Using the code as the base of a commercial product
* Redistributing it as your own work
* Financially exploiting it without explicit authorization from the author

Allowed:

* Reading, studying, and citing it as a reference
* Mentioning it with attribution to the original author

---

## Usage note

When adding a new feature, review this table before designing it. If it benefits Rodrigo but complicates Diego’s experience, it’s a signal that an additional abstraction layer is needed.
