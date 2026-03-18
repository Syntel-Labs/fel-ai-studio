# PDF Generation — FEL AI Studio

## Responsibility

This module transforms a FEL invoice XML into a visually customized PDF document. It is the layer closest to the system’s final output and the one directly perceived by the user.

Generation always happens on demand, never in the background.

The result is not persisted on the server: in web mode it is deleted after download, in Docker mode it is written directly to the user’s mounted volume.

---

## Stack

* **ReportLab** — programmatic PDF generation
* **Python** — layout and composition logic
* **ElementTree** — FEL XML parsing (stdlib, no external dependencies)

---

## What `readFelXml` does

Parses the FEL XML and extracts the fields required to build the PDF.

| Field                 | Description                                       |
| --------------------- | ------------------------------------------------- |
| `numero_autorizacion` | SAT authorization UUID                            |
| `nit`                 | Issuer NIT                                        |
| `nombre_emisor`       | Issuer name or business name                      |
| `id_receptor`         | Recipient NIT or CUI                              |
| `nombre_receptor`     | Recipient name or business name                   |
| `fecha`               | Issue date                                        |
| `serie`               | DTE series                                        |
| `numero_dte`          | DTE number                                        |
| `moneda`              | Currency (default GTQ)                            |
| `forma_pago`          | Payment method                                    |
| `subtotal`            | Amount before VAT                                 |
| `iva`                 | Calculated VAT (12%)                              |
| `total`               | Total amount                                      |
| `monto`               | Alternative amount field (used in validation)     |
| `items`               | List of line items (description, quantity, price) |

**Note:** FEL XML uses SAT-specific namespaces. The parser handles them explicitly with `ElementTree` to avoid breaking extraction. The original filename is always discarded: the system works with an internally assigned UUID path.

---

## What `generatePdf` does

Takes the extracted XML data and the `PdfConfig` visual configuration, and builds the PDF using direct ReportLab Canvas.

### Target signature

```python
generatePdf(
    xmlPath: str,
    outputPdf: str,
    config: PdfConfig,
)
```

`xmlPath` is always an absolute path validated against the session sandbox.

`outputPdf` is always a unique temporary path per session in web mode, or a path inside the mounted volume in Docker mode.

---

## PdfConfig — full structure

```python
@dataclass
class PdfConfig:
    # 7 semantic color tokens
    primary:        str
    primary_text:   str
    secondary:      str
    secondary_text: str
    text_main:      str
    text_soft:      str
    background:     str

    # Identity
    logo_path: str | None
    pdf_name:  str

    # Footer
    website: str = ""
    phone:   str = ""
    email:   str = ""

    # Layout
    top_bar_height: int = 20
    qr_size:        int = 150
```

### Dynamic variables in `pdf_name`

The generated PDF filename supports variables extracted from the XML:

| Variable  | Replaced with         |
| --------- | --------------------- |
| `{serie}` | DTE series            |
| `{dte}`   | DTE number            |
| `{fecha}` | Issue date (YYYYMMDD) |
| `{nit}`   | Issuer NIT            |

Example: `factura_{serie}_{dte}.pdf` → `factura_9E52C854_3784000961.pdf`

---

## How PdfConfig is built by mode

### Web mode

`PdfConfig` is built from the decoded JWT payload on each request.

All 7 color tokens, `logo_ref`, `pdf_name`, and footer data come from the JWT signed in the configuration panel. Nothing is read from `.env`.

```bash
JWT payload → FastAPI middleware → PdfConfig → generatePdf()
```

### Docker Hub mode

`PdfConfig` follows this precedence hierarchy:

```bash
explicit arguments in tool call (prompt override)
        │  overrides
        ▼
/app/data/fel_config.json (user config)
        │  overrides
        ▼
container environment variables (optional fallback)
        │  overrides
        ▼
system default values
```

If `fel_config.json` does not exist, the server starts with defaults and logs it. It does not fail: it degrades gracefully.

---

## PDF layout

Based on the reference PDF from the previous implementation:

```bash
┌─────────────────────────────────────────┐
│  [top bar — primary / optional]         │
├───────────┬─────────────────────────────┤
│           │  Series / DTE Number        │
│   LOGO    │  Issuer name and NIT        │
│           │  (right aligned)            │
├───────────┴─────────────────────────────┤
│  Recipient name                        │
│  Recipient NIT                         │
│  Date / Currency                       │
├─────────────────────────────────────────┤
│  Items table                           │
│  [header — primary / primary_text]     │
│  Concept | Qty | Price | Total         │
│  [rows — text_soft]                    │
├─────────────────────────────────────────┤
│  Subtotal                              │
│  VAT 12%                               │
│  Total [text_main — bold]              │
├──────────────────────┬──────────────────┤
│  Payment method      │                  │
│  Notes / retention   │   QR code        │
│  Authorization no.   │                  │
│  Certifier / NIT     │                  │
├──────────────────────┴──────────────────┤
│  [footer — secondary]                  │
│  🌐 website   📞 phone   ✉ email       │
│  [icons + text — secondary_text]       │
└─────────────────────────────────────────┘
```

**Top bar:** configurable height (`top_bar_height`). Can be set to 0 to remove it visually.

**QR code:** encodes the SAT authorization number. Generated programmatically using `qrcode`, not as an external asset.

**Footer icons:** rendered as Unicode characters or simple ReportLab symbols. No local assets (consistent with ADR 0004).

---

## Fonts

Helvetica is used as the base ReportLab font. No local TTF files or manual font registration (consistent with ADR 0004).

| Variant           | Usage                          |
| ----------------- | ------------------------------ |
| Helvetica         | General text, values           |
| Helvetica-Bold    | Labels, headers, Total         |
| Helvetica-Oblique | Notes, optional secondary text |

Typography customization is a future extension (v2).

---

## Color application

The 7 tokens are received as hex strings and converted to ReportLab `HexColor` objects at generation time:

```python
from reportlab.lib.colors import HexColor

primary        = HexColor(config.primary)
primary_text   = HexColor(config.primary_text)
secondary      = HexColor(config.secondary)
secondary_text = HexColor(config.secondary_text)
text_main      = HexColor(config.text_main)
text_soft      = HexColor(config.text_soft)
background     = HexColor(config.background)
```

---

## Validation before generation

Before `generatePdf`, the system runs `fel_validate` via MCP.

If it returns `ok=False`, Claude evaluates the issues and decides whether to proceed or inform the user (Variant 1, ADR 0002).

PDFs may still be generated with minor validation issues. Claude summarizes them in natural language.

---

## Output by mode

### Web mode

```bash
/tmp/fel/<session_uuid>/out_<file_uuid>.pdf
  └── StreamingResponse with Content-Disposition: attachment
  └── Filename = pdf_name with resolved variables
  └── File deleted immediately after sending
```

### Docker mode

```bash
/app/data/out/<resolved_pdf_name>.pdf
  └── Written directly to user-mounted volume
  └── Not deleted: remains in local folder
```

---

## Batch

`fel_batch` reuses `generatePdf` in a loop over all `*.xml` files in a directory. Produces one PDF per file and a `manifest.json` mapping `xml → pdf`.

In web mode, batch is invoked from chat (no dedicated UI).

In Docker mode, it is the natural way to process invoices in bulk from the mounted volume.

`PdfConfig` follows the same precedence hierarchy as single render.

---

## Notes resolved in this document

* **QR library:** `qrcode` confirmed — generates images embeddable in ReportLab via `qrcode.make()` → `ImageReader`. `python-barcode` is for linear barcodes, not QR. Discarded.

* **`top_bar_height=0`:** fully hides the top bar. Valid value. Layout shifts accordingly without error.

* **Custom fonts (v2):** planned extension.
  `PdfConfig` reserves `font_path: str | None = None`.
  When implemented, enabled via new ADR extending session model (JWT or `fel_config.json`).

---

## Related files

* `adr/0002-mcp-as-orchestration-layer.md` → who decides when to render
* `adr/0004-no-local-fonts-or-icons.md` → fonts and icons policy
* `adr/0006-docker-hub-mcp-distribution.md` → PdfConfig precedence in Docker
* `technical/mcp-server.md` → how fel_render and fel_batch expose this logic
* `technical/session-model.md` → how PdfConfig comes from JWT in web mode
* `technical/engine.md` → how ChatEngine invokes fel_render via MCP
