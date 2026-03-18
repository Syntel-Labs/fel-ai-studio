# Config Panel — FEL AI Studio

## Responsibility

The configuration panel is where the user defines their visual identity, enters their API Key, and generates the session link. It is the only place in the application where the JWT is created and where visual configuration is established for the entire session.

---

## General layout

```bash id="6l2lcs"
┌─────────────────────────────────────────────────────────┐
│  ⚙ Configuration                                        │
├──────────────────────────┬──────────────────────────────┤
│                          │                              │
│   Left panel             │   Right preview              │
│   (form)                 │   (real-time PDF)            │
│                          │                              │
│   1. API Key             │   ┌──────────────────────┐   │
│   2. Visual identity     │   │ [top bar]            │   │
│      └ Colors            │   │ LOGO │ issuer data   │   │
│      └ Logo              │   │      │ recipient data│   │
│   3. Document            │   │ items table          │   │
│      └ PDF name          │   │ subtotal / VAT /total│   │
│      └ Footer            │   │ [footer]   │  QR     │   │
│   4. Layout              │   └──────────────────────┘   │
│      └ Top bar           │                              │
│      └ QR size           │   Expires: 17/03/2025 11:23  │
│                          │   [— Copy link —]            │
│   [Generate link]        │   [↺ Regenerate]             │
│                          │                              │
└──────────────────────────┴──────────────────────────────┘
```

On small screens, the preview collapses below the form.

The “Generate link” button is always visible without scrolling.

---

## Section 1 — API Key

### Input field

```bash id="4o7yz9"
┌─────────────────────────────────────────┐
│ Anthropic API Key                       │
│ ┌─────────────────────────────────┬───┐ │
│ │ sk-ant-••••••••••••••••••••••   │ 👁 │ │
│ └─────────────────────────────────┴───┘ │
│ ✓ Valid format                         │
│                                         │
│ ℹ Your key never leaves this browser.   │
│   Get yours at console.anthropic.com    │
└─────────────────────────────────────────┘
```

**Behavior:**

* Password-type input by default, with visibility toggle
* Real-time format validation: expected prefix `sk-ant-`
* Visual indicator: ✓ green if valid, ✗ red if not
* No backend validation: client-side format only
* Stored in `sessionStorage` on blur or Enter
* Cleared from `sessionStorage` when tab closes

**Field states:**

| State          | Indicator | Button effect   |
| -------------- | --------- | --------------- |
| Empty          | None      | Button disabled |
| Invalid format | ✗ red     | Button disabled |
| Valid format   | ✓ green   | Button enabled  |

**Helper message:**

> "Your API Key never leaves this browser or is stored on the server.
> It is used only while the session is active."

---

## Section 2 — Visual identity

### 2a. Colors — 7 semantic tokens

Each token includes:

* Semantic name
* Description of usage
* Color swatch
* Editable hex input
* Native color picker

```bash id="iqmp30"
┌─────────────────────────────────────────┐
│ HEADER                                  │
├─────────────────────────────────────────┤
│ Top bar background                      │
│ Applied to: header bar, table header    │
│ [████████] #fe5101  [🎨]               │
├─────────────────────────────────────────┤
│ Top bar text                            │
│ Applied to: header text, table header   │
│ [████████] #ffffff  [🎨]               │
├─────────────────────────────────────────┤
│ BODY                                    │
├─────────────────────────────────────────┤
│ Document background                     │
│ Applied to: full PDF background         │
│ [████████] #ffffff  [🎨]               │
├─────────────────────────────────────────┤
│ Main text                               │
│ Applied to: bold labels, Total row      │
│ [████████] #000000  [🎨]               │
├─────────────────────────────────────────┤
│ Secondary text                          │
│ Applied to: values, table rows          │
│ [████████] #8c8d8e  [🎨]               │
├─────────────────────────────────────────┤
│ FOOTER                                  │
├─────────────────────────────────────────┤
│ Footer background                       │
│ Applied to: bottom bar                  │
│ [████████] #343534  [🎨]               │
├─────────────────────────────────────────┤
│ Footer text and icons                   │
│ Applied to: footer text and icons       │
│ [████████] #ffffff  [🎨]               │
└─────────────────────────────────────────┘
```

**Behavior:**

* Changes update preview instantly
* Hex input accepts with or without `#`
* Invalid hex shows red border and does not update preview
* “Reset defaults” restores base palette

**Default palette:**

| Token            | Default   |
| ---------------- | --------- |
| `primary`        | `#fe5101` |
| `primary_text`   | `#ffffff` |
| `secondary`      | `#343534` |
| `secondary_text` | `#ffffff` |
| `text_main`      | `#000000` |
| `text_soft`      | `#8c8d8e` |
| `background`     | `#ffffff` |

---

### 2b. Logo

```bash id="t1tahg"
┌─────────────────────────────────────────┐
│ Company logo                            │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │                                 │    │
│  │   Drag your logo here           │    │
│  │   or click to select            │    │
│  │                                 │    │
│  │   JPG, PNG, WEBP — max 200KB    │    │
│  └─────────────────────────────────┘    │
│                                         │
│  [ No logo — PDF generated without it ] │
└─────────────────────────────────────────┘
```

**Behavior:**

* Drag & drop or click upload
* Preview shown after upload
* Validation: extension and size
* Upload immediately via `POST /logo`
* Stores `logo_ref` locally
* “Remove logo” clears state and preview

---

## Section 3 — Document

### 3a. PDF name

```bash id="jdb3nb"
┌─────────────────────────────────────────┐
│ PDF file name                           │
│ ┌─────────────────────────────────────┐ │
│ │ factura_{serie}_{dte}.pdf           │ │
│ └─────────────────────────────────────┘ │
│                                         │
│ Available variables:                    │
│  {serie}, {dte}, {fecha}, {nit}         │
│                                         │
│ Example: factura_9E52C854_3784000961.pdf│
└─────────────────────────────────────────┘
```

**Behavior:**

* Free text input with dynamic variables
* Live example preview
* Invalid filename characters auto-removed
* Defaults restored if empty

---

### 3b. Footer

```bash id="2n7f0i"
┌─────────────────────────────────────────┐
│ Footer contact info                     │
│                                         │
│ Website                                 │
│ ┌─────────────────────────────────────┐ │
│ │ https://miempresa.com               │ │
│ └─────────────────────────────────────┘ │
│                                         │
│ Phone                                   │
│ ┌─────────────────────────────────────┐ │
│ │ (502) 2254-9885                     │ │
│ └─────────────────────────────────────┘ │
│                                         │
│ Email                                   │
│ ┌─────────────────────────────────────┐ │
│ │ info@miempresa.com                  │ │
│ └─────────────────────────────────────┘ │
│                                         │
│ [ Leave empty — no footer data ]        │
└─────────────────────────────────────────┘
```

**Behavior:**

* All fields optional
* Empty → only background shown
* Live preview updates
* Checkbox clears and disables fields

---

## Section 4 — Layout

### 4a. Top bar height

```bash id="sfl9ct"
┌─────────────────────────────────────────┐
│ Top bar                                │
│ Height: [──────●──────────] 20px       │
│ 0px = hidden   max = 60px              │
└─────────────────────────────────────────┘
```

### 4b. QR size

```bash id="rq33j4"
┌─────────────────────────────────────────┐
│ QR code                                │
│ Size: [────────●────────] 150px        │
│ 80px = small   max = 200px             │
└─────────────────────────────────────────┘
```

---

## PDF preview

### Behavior

Rendered as a React component (SVG/CSS), not via backend.

* Uses real config values
* Uses mock invoice data

```bash id="25wq09"
Example company: CINCO CONSULTORES, S.A.
Subtotal: Q8,010.59
VAT: Q961.27
Total: Q8,971.86
```

**Reason:** preview must work without XML.

---

## Generated link area

```bash id="4c6bq8"
┌─────────────────────────────────────────┐
│ ✓ Link generated                        │
│ https://app/chat?session=eyJhbGc...     │
│ [📋 Copy]                               │
│                                         │
│ Expires: Mar 17, 2025, 11:23            │
│ ████████████████░░░░ 58 min remaining   │
│                                         │
│ [↺ Regenerate link]                     │
└─────────────────────────────────────────┘
```

**Behavior:**

* Copy button → clipboard
* Countdown progress bar (updates every 30s)
* Last 5 minutes → orange + pulse animation
* Regenerate replaces JWT

---

## Global panel state

Stored in `sessionStorage`:

| Key                   | Content              |
| --------------------- | -------------------- |
| `fel_api_key`         | User API Key         |
| `fel_colors`          | Color tokens         |
| `fel_logo_ref`        | Logo UUID            |
| `fel_pdf_name`        | PDF template         |
| `fel_footer`          | Footer data          |
| `fel_layout`          | Layout config        |
| `fel_session_token`   | JWT                  |
| `fel_session_expires` | Expiration timestamp |

**Reload:** state rehydrated
**Tab close:** state cleared

---

## Full panel flow

```bash id="d1yoyr"
1. Open CONFIG
2. Enter API Key
3. Configure colors
4. Upload logo (optional)
5. Configure document + footer
6. Adjust layout
7. Click "Generate link"
   └── POST /session
   └── Store token
   └── Show link + expiration
8. Copy link → go to CHAT
```

---

## Resolved notes

* Preview implemented as React-rendered SVG (no backend calls)
* Countdown animation via CSS + setInterval
* Mobile: sidebar collapses into overlay, actions always visible

---

## Related files

* `ui/ux-decisions.md` → UX principles
* `technical/session-model.md` → JWT structure
* `technical/api-reference.md` → `/session`, `/logo`
* `adr/0003-user-owned-api-key.md` → API Key design
* `adr/0005-ephemeral-link-generation.md` → link generation
* `adr/0004-no-local-fonts-or-icons.md` → icons/fonts policy
