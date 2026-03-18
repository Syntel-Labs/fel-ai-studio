# ADR 0004 — No Local Fonts or Icons

**Status:** Accepted

**Date:** 2025-03-17

---

## Context

The previous code included local typography (Montserrat, RobotoMono) manually registered in ReportLab, and icons referenced from the container’s filesystem. These assets lived in `assets/fonts/` and had to be present in the repo and in the container for the system to work.

When moving to a portfolio- and product-oriented architecture, this model presents several problems:

* It increases repository size with binaries (`.ttf` files)
* It introduces an implicit dependency that is not evident until the system fails in a new environment
* Local fonts are not configurable by the user from the interface
* It complicates the Docker container build without adding differential value

The question is: how does the system handle typography and iconography in this version?

---

## Scope of this decision

This decision applies to two distinct contexts:

1. **Frontend (React + Vite):** icons and typography of the user interface
2. **Backend (ReportLab):** fonts embedded in the generated PDF

Both are solved consistently: no mandatory local assets in v1.

---

## Options considered — Frontend

### Option A — Icons and fonts as local assets

SVGs downloaded into the repo, fonts in `public/fonts/`.

**Advantage:** works offline, no external runtime dependencies.

**Disadvantage:** additional repo weight, manual updates.

**Verdict:** discarded.

---

### Option B — CDN at runtime (Google Fonts, Lucide) ✓

Fonts from Google Fonts via `@import`. Icons from Lucide installed as an npm dependency (tree-shakeable).

**Advantage:** clean repo, no binaries, standard in modern React projects.

**Disadvantage:** requires internet connection on first load.

**Verdict:** accepted for v1.

---

### Option C — Full design system (MUI, Chakra, etc.)

**Advantage:** guaranteed visual consistency.

**Disadvantage:** significant overhead, hides intentional design decisions behind third-party abstractions.

**Verdict:** discarded. The frontend is built with Tailwind + Lucide.

---

## Options considered — Backend (ReportLab / PDF)

### Option A — Local TTF fonts (previous model)

`assets/fonts/Montserrat/` and `assets/fonts/Roboto_Mono/` in the repo.

**Advantage:** full control over PDF typography.

**Disadvantage:** binaries in the repo, the container fails silently if files are not in the expected location. Not configurable by the user.

**Verdict:** discarded.

---

### Option B — Standard ReportLab fonts (Helvetica)

ReportLab includes 14 base fonts (Type 1) with no external files. Helvetica as the main font.

**Advantage:** zero asset dependencies, reproducible in any environment.

**Disadvantage:** less distinctive typography than Montserrat.

**Verdict:** accepted as the default for v1.

---

### Option C — Optional custom fonts via configuration

The user uploads a TTF font from the configuration panel.

**Advantage:** maximum flexibility.

**Disadvantage:** adds complexity to the configuration flow, session model, and MCP server.

**Verdict:** discarded for v1. Documented as a future extension.

---

## Decision

**The system does not include fonts or icons as mandatory local assets in v1.**

* **Frontend:** Google Fonts via CDN + Lucide as an npm dependency
* **Backend / PDF:** Helvetica as the base ReportLab font
* **Icons in PDF:** none in v1. The QR code is generated programmatically

---

## Consequences

**Positive:**

* The repository contains no binaries
* The Docker container is reproducible with just `docker-compose up`
* Applies equally to web mode and standalone Docker Hub mode

**Negative:**

* PDFs in v1 use Helvetica, not a branded font
* The frontend requires connection to Google Fonts CDN on first load

**Neutral:**

* `ACTIVE_FONT`, `FONT_DIR_MONTSERRAT`, `FONT_DIR_ROBOTOMONO` are removed from `config.py` in the rewrite
* The fonts section in `PdfConfig` is reserved for v2 with `None` default

---

## Constraints imposed by this decision

* No `pdfmetrics.registerFont()` is executed at MCP server startup
* No `assets/fonts/` directories are created in the repo
* The `Dockerfile` does not copy or download TTF fonts
* If custom font support is added, it must be via a new ADR that extends `PdfConfig` and the session model

---

## Future review

In v2, backend Option C can be implemented without affecting this decision: there would still be no mandatory local fonts in the repo.

---

## Related files

* `adr/0001-stateless-session-model.md` → session context does not include fonts in v1
* `technical/pdf-generation.md` → PdfConfig and available fonts
* `architecture/stack.md` → Lucide and Google Fonts as frontend dependencies
