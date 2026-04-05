# StrikeLog
An in-house pentest report templating tool designed to help gain experience in writing professional grade reports. This was made in my off-time to practice writing reports, specifically for HackTheBox machines and other platforms. It is NOT used professionally, but rather a training tool designed to focus on learning how to write them and not worry about the overall design.

<p align="center">
  <img src="https://raw.githubusercontent.com/r0otsec/StrikeLog/refs/heads/main/imgs/logo.png" width=500>
</p>

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Directory Structure](#directory-structure)
- [Quick Start](#quick-start)
  - [Docker (Recommended)](#docker-recommended)
  - [Local Development](#local-development)
- [PDF Generator (`pentest-report/`)](#pdf-generator-pentest-report)
  - [How It Works](#how-it-works)
  - [CLI Usage](#cli-usage)
  - [Report Data Format (YAML)](#report-data-format-yaml)
  - [PDF Generation Pipeline](#pdf-generation-pipeline)
  - [Report Sections](#report-sections)
- [Web Application (StrikeLog)](#web-application-strikelog)
  - [Features](#features)
  - [Editor Tabs](#editor-tabs)
  - [Finding Editor](#finding-editor)
  - [Content Block System](#content-block-system)
  - [API Reference](#api-reference)
- [Customisation](#customisation)
  - [Branding & Logo](#branding--logo)
  - [Report Templates (Jinja2)](#report-templates-jinja2)
  - [Styling (CSS)](#styling-css)
  - [Adding a New Report Section](#adding-a-new-report-section)
- [Data Schema Reference](#data-schema-reference)
- [Security Notes](#security-notes)

---

## Overview

StrikeLog has two layers that work together:

| Layer | Location | Purpose |
|-------|----------|---------|
| **PDF Generator** | `pentest-report/` | Standalone Python tool that takes structured data (YAML or dict) and renders a professional PDF using Jinja2 templates + Playwright/Chromium |
| **Web Application** | `web/` | Full-stack browser editor (FastAPI + React) for creating engagements, writing findings, uploading screenshots, and triggering PDF generation |

The two layers are deliberately decoupled. The web app calls `generate.py` via a Python import bridge meaning the PDF generator can also be used entirely from the command line without the web UI.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Browser (React SPA)                          │
│                                                                     │
│  Dashboard → Engagement Editor → Finding Drawer → Generate PDF      │
│      │              │                  │               │            │
│      └──────────────┴──────────────────┴───────────────┘            │
│                           API calls (/api/*)                        │
└────────────────────────────┬────────────────────────────────────────┘
                             │ HTTP (port 8080)
┌────────────────────────────▼────────────────────────────────────────┐
│                    FastAPI Backend (web/backend/)                    │
│                                                                     │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │  Auth / JWT │  │ Project CRUD │  │ Media Upload │               │
│  │  (bcrypt)   │  │  (SQLAlchemy)│  │  /api/media/ │               │
│  └─────────────┘  └──────────────┘  └──────────────┘               │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │            POST /api/projects/{id}/generate                  │   │
│  │                                                              │   │
│  │  report_data (JSON) ──► run_in_executor ──► generate.py     │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  SQLite (web/data/reports.db)   Media (web/data/media/{project}/)   │
└────────────────────────────┬────────────────────────────────────────┘
                             │ Python import (sys.path)
┌────────────────────────────▼────────────────────────────────────────┐
│              PDF Generator (pentest-report/generate.py)             │
│                                                                     │
│  process_report()          render_html()          generate_pdf()    │
│       │                        │                       │            │
│  • Risk counts             Jinja2 render          • Pass 1: full   │
│  • Severity sorting         → HTML string           doc + header   │
│  • Markdown → HTML                                • Pass 2: cover  │
│  • Scope normalisation    templates/               only (no header)│
│  • Screenshot → base64    ├── report.html.j2      • pypdf merge    │
│  • Figure numbering       └── partials/            (preserves PDF  │
│  • Code highlighting          _cover.html.j2       link annots)    │
│                               _toc.html.j2                         │
│                               _introduction.html.j2                │
│                               _executive_summary.html.j2           │
│                               _finding_overview.html.j2            │
│                               _finding.html.j2                     │
│                               _exploitation_scenarios.html.j2      │
│                               _remediation_roadmap.html.j2         │
│                               _appendices.html.j2                  │
│                                                                     │
│                          assets/style.css                           │
└─────────────────────────────────────────────────────────────────────┘
```

### PDF Generation Flow

```
Browser clicks "Generate PDF"
        │
        ▼
POST /api/projects/{id}/generate
        │
        ▼
Read project.report_data from SQLite
        │
        ▼
loop.run_in_executor()          ← runs in thread pool (Playwright is sync-only)
        │
        ▼
generate_pdf_from_dict(data, media_dir)
        │
        ├─► process_report(data)
        │       ├── compute risk counts + generate SVG bar chart
        │       ├── sort findings by severity
        │       ├── Markdown → HTML for all narrative fields
        │       ├── normalise scope format (web → template)
        │       ├── process content_blocks[] (text/code/screenshot)
        │       ├── encode screenshots as base64 data URIs
        │       ├── assign sequential figure numbers
        │       └── load company logo as base64
        │
        ├─► render_html(report)      ← Jinja2 → full HTML string
        │
        ├─► Pass 1: html_to_pdf_playwright()
        │       └── full document + running header/footer → body.pdf
        │
        ├─► Pass 2: html_to_pdf_playwright()
        │       └── cover page only, no header → cover.pdf
        │
        ├─► _merge_cover_with_body()
        │       └── pypdf PdfWriter.append() → final.pdf
        │           (preserves internal link annotations)
        │
        └─► StreamingResponse(pdf_bytes)
                │
                ▼
        Browser downloads PDF
```

---

## Directory Structure

```
RootSec-PDFs/
│
├── README.md                            ← you are here
├── requirements.txt                     ← PDF generator Python deps
│
├── pentest-report/                      ← standalone PDF generator
│   ├── generate.py                      ← all logic: processing, rendering, Playwright
│   ├── data/
│   │   └── example_report.yaml         ← fully annotated example — copy & edit
│   └── templates/
│       ├── report.html.j2              ← master template (includes all partials)
│       ├── assets/
│       │   └── style.css               ← all PDF CSS (A4, dark-themed, print-ready)
│       └── partials/
│           ├── _cover.html.j2
│           ├── _toc.html.j2
│           ├── _introduction.html.j2
│           ├── _executive_summary.html.j2
│           ├── _technical_summary.html.j2
│           ├── _finding_overview.html.j2
│           ├── _exploitation_scenarios.html.j2
│           ├── _assessment_section.html.j2
│           ├── _finding.html.j2
│           ├── _remediation_roadmap.html.j2
│           └── _appendices.html.j2
│
├── screenshots/
│   └── logo.png                        ← RootSec firm logo (appears in page header + cover)
│
└── web/
    ├── run.py                           ← entry point: sets up sys.path, starts uvicorn
    ├── docker-compose.yml
    ├── Dockerfile                       ← multi-stage: Node build + Playwright runtime
    ├── data/
    │   ├── reports.db                   ← SQLite (auto-created on first run)
    │   └── media/{project_id}/          ← uploaded screenshots per engagement
    │
    ├── backend/
    │   ├── main.py                      ← FastAPI app + all API routes
    │   ├── models.py                    ← SQLAlchemy models
    │   ├── database.py                  ← async engine, session, additive migrations
    │   ├── auth.py                      ← JWT + bcrypt auth
    │   └── requirements.txt
    │
    └── frontend/
        ├── src/
        │   ├── App.tsx                  ← router, AuthProvider, ProtectedRoute
        │   ├── auth.tsx                 ← AuthContext + useAuth() hook
        │   ├── api.ts                   ← all API calls centralised here
        │   ├── types.ts                 ← TypeScript interfaces for all data
        │   ├── components/
        │   │   ├── ContentBlockEditor.tsx   ← block-based evidence editor
        │   │   ├── FindingDrawer.tsx        ← slide-out finding editor panel
        │   │   ├── SeverityBadge.tsx
        │   │   ├── Modal.tsx
        │   │   └── Sidebar.tsx
        │   └── pages/
        │       ├── Login.tsx
        │       ├── Dashboard.tsx            ← engagement list
        │       ├── ProjectDetail.tsx        ← main engagement editor (all tabs)
        │       ├── FindingsLibrary.tsx      ← reusable finding templates
        │       ├── Clients.tsx
        │       ├── Domains.tsx
        │       ├── AdminUsers.tsx
        │       └── ReportDesigns.tsx
        ├── package.json
        ├── tailwind.config.js
        └── vite.config.ts
```

## Quick Start

### Docker (Recommended)

The Docker image bundles the React frontend, FastAPI backend, and a full Playwright/Chromium installation for PDF generation.

```bash
git clone https://github.com/rootsec/strikelog.git
cd strikelog/web

docker compose up -d
```

Open **http://localhost:8080** in your browser.

Default credentials (change immediately):
- **Username:** `admin`
- **Password:** `admin123`

> Data is persisted in the named Docker volume `rootsec-data`. Your engagements, uploads, and database survive container restarts and rebuilds.

To rebuild after pulling updates:
```bash
docker compose down
docker compose up -d --build
```

> [!WARNING]  
> Currently, the docker compose way of running this tool is broken. Please use the local development instead for now.

### Local Development

Requires Python 3.11+, Node.js 20+.

**Terminal 1 Backend:**
```bash
cd web
pip install -r backend/requirements.txt
pip install playwright && playwright install chromium
python run.py
# Backend runs on http://localhost:8080
```

**Terminal 2 Frontend (hot reload):**
```bash
cd web/frontend
npm install
npm run dev
# Frontend dev server on http://localhost:5173
# Vite proxies /api/* → http://localhost:8080
```

**PDF generator only (no web UI):**
```bash
pip install -r requirements.txt
playwright install chromium
cd pentest-report
python generate.py data/example_report.yaml
python generate.py data/example_report.yaml --output report.pdf --open
```

## PDF Generator (`pentest-report/`)

### How It Works

`generate.py` is a self-contained Python script that:

1. **Loads** a YAML file (CLI) or receives a Python dict (web API)
2. **Processes** the data, sorts findings by severity, converts Markdown to HTML, normalises scope format, base64-encodes screenshots, assigns sequential figure numbers, generates a risk bar chart SVG
3. **Renders** the processed data through Jinja2 HTML templates
4. **Generates** a PDF via a two-pass Playwright/Chromium process (see below)
5. **Returns** the PDF bytes

### CLI Usage

```bash
# Basic usage
python generate.py data/my_report.yaml

# Specify output filename
python generate.py data/my_report.yaml --output RS-2026-001-ClientName.pdf

# Preview HTML in browser before generating PDF
python generate.py data/my_report.yaml --html

# Generate and open PDF immediately
python generate.py data/my_report.yaml --open
```

### Report Data Format (YAML)

The full reference is `pentest-report/data/example_report.yaml`. The top-level keys are:

```yaml
meta:                    # Report metadata, client & consultant info, dates
version_history:         # Document control table (Appendix B)
introduction:            # Preamble, objective, approach, testing team, dates, caveats, authorisation
scope:                   # In-scope targets, out-of-scope, methodology/standards
executive_summary:       # Overall risk rating, intro narrative, key findings (Management Summary)
technical_summary:       # Technical narrative (subsection under Executive Summary)
risk_summary:            # Auto-computed from findings; override counts manually if needed
sections:                # Assessment sections, each containing findings[]
exploitation_scenarios:  # Multi-step attack chains / significant findings
remediation_roadmap:     # Manual timeframe overrides (auto-generated from severity if empty)
tools_used:              # Appendix A — tools & utilities
appendices:              # Reserved for future custom appendices
```

#### Minimal Finding Example

```yaml
sections:
  - id: internal-infrastructure
    title: "Internal Infrastructure"
    enabled: true
    findings:
      - id: INT-001
        title: "Kerberoastable Service Accounts"
        severity: critical           # critical | high | medium | low | informational
        description: |
          Multiple service accounts were found to be configured with weak,
          crackable passwords and were vulnerable to Kerberoasting...
        risk_rating_justification: |
          CVSS 4.0 score of 9.1 (Critical). Direct path to domain compromise.
        impact:
          technical: |
            An attacker with domain user access can request Kerberos service tickets
            for these accounts and crack them offline, gaining plaintext credentials.
          business: |
            Successful exploitation leads to full domain administrator access and
            the ability to exfiltrate all data within the environment.
        recommendations:
          tactical:
            - Immediately rotate passwords for all identified service accounts to
              random 25+ character strings
          strategic:
            - Implement Group Managed Service Accounts (gMSA) for all service accounts
        affected_hosts:
          - "svc_backup@corp.local"
          - "svc_sql@corp.local"
        references:
          - title: "Kerberoasting — MITRE ATT&CK T1558.003"
            url: "https://attack.mitre.org/techniques/T1558/003/"
        evidence:
          - type: screenshot
            path: "screenshots/kerberoast_hashes.png"
            caption: "Service ticket hashes captured via Rubeus"
          - type: code
            language: bash
            label: "Rubeus — Kerberoasting"
            content: |
              Rubeus.exe kerberoast /outfile:hashes.txt /format:hashcat
            highlight_lines: [1]
```

### PDF Generation Pipeline

StrikeLog uses a **two-pass Playwright + pypdf merge** strategy to produce a clean PDF where:
- The **cover page** is full-bleed dark with no running header
- All **subsequent pages** carry the RootSec logo header and a footer with client name, report title, and page number

**Why two passes?** Playwright's `margin` parameter sets the page margin globally and overrides CSS `@page` rules. There is no reliable CSS mechanism to suppress the header on only the first page. The solution:

| Pass | Input | Output | Header? |
|------|-------|--------|---------|
| 1 | Full HTML document | `body.pdf` (all pages, with header) | Yes |
| 2 | Cover-only HTML | `cover.pdf` (1 page, no header) | No |
| Merge | cover.pdf page 0 + body.pdf pages 1+ | `final.pdf` | Cover clean, rest have header |

The merge uses `PdfWriter.append()` (pypdf 4.x) rather than `add_page()` to correctly remap internal GoTo link annotations across the two source PDFs — this is what makes TOC and findings table links clickable in the final PDF.

**TOC page numbers** are injected via a JavaScript `page.evaluate()` call before `page.pdf()` fires. The script counts page-breaking elements (`.cover-page`, `.toc-page`, `.report-section`, `.finding-card`) in DOM order and stamps each `.toc-row` with the correct page number.

### Report Sections

Sections appear in this fixed order in the PDF:

| # | Section | Template |
|---|---------|----------|
| 1 | Cover Page | `_cover.html.j2` |
| 2 | Table of Contents | `_toc.html.j2` |
| 3 | Introduction | `_introduction.html.j2` |
| 4 | Executive Summary (+ Management & Technical sub-sections) | `_executive_summary.html.j2` |
| 5 | Technical Findings Overview (risk chart + table) | `_finding_overview.html.j2` |
| 6 | Exploitation Scenarios *(if any)* | `_exploitation_scenarios.html.j2` |
| 7 | Assessment Sections → individual findings | `_assessment_section.html.j2` + `_finding.html.j2` |
| 8 | Remediation Roadmap | `_remediation_roadmap.html.j2` |
| 9 | Appendix A — Tools & Utilities *(if tools_used is non-empty)* | `_appendices.html.j2` |
| 10 | Appendix B — Engagement Information | `_appendices.html.j2` |
| 11 | Appendix C — Risk Rating Methodology | `_appendices.html.j2` |

> **Note:** If `tools_used` is empty, Appendix A is omitted and the letters shift: Engagement Information becomes A, Risk Rating Methodology becomes B.

## Web Application (StrikeLog)

### Features

- **Engagement management**: create, archive, and track penetration test engagements linked to clients
- **Structured finding editor**: severity badges, affected hosts, references, recommendations, risk rating justification, retest tracking
- **Block-based evidence**: interleave narrative text, syntax-highlighted code blocks, and screenshots in any order within a finding
- **Image resize & PDF preview**: drag a handle to set image width as a percentage; a live white-page preview shows exactly how it will appear centred in the PDF
- **Exploitation scenarios**: document multi-step attack chains with the same block editor
- **Remediation roadmap**: auto-generated from finding severities, or manually override per-tier (Immediate / Short / Medium / Long)
- **Finding templates library**: save and re-use boilerplate findings across engagements
- **Client & domain management**: maintain client contacts and track infrastructure/domains
- **User management**: admin-only user CRUD, role-based access (admin / user)
- **One-click PDF**: generate and download the full report PDF without leaving the browser
- **Auto-save**: all changes debounced and saved to the backend automatically (1.5s delay)

### Editor Tabs

The main engagement editor (`ProjectDetail.tsx`) is organised into tabs:

| Tab | What you edit |
|-----|--------------|
| **Overview** | Report ID, title, subtitle, classification, dates, client details, consultant details, version history |
| **Introduction** | Preamble, engagement objective, assessment approach, in-scope targets (category + hosts), out-of-scope list, methodology & standards, testing team, testing dates, caveats, authorisation statement |
| **Findings** | Assessment sections and the findings within each. Click any finding row to open the slide-out editor |
| **Scenarios** | Exploitation scenarios / significant attack chains. Each scenario uses the same block-based content editor |
| **Summary** | Overall risk rating, executive summary intro narrative, management summary key findings (bullet list), technical summary narrative |
| **Roadmap** | Manual remediation roadmap w/ 4 tiers: Immediate (0–7d), Short-Term (8–30d), Medium-Term (31–90d), Long-Term (90d+). Leave empty to use the auto-generated roadmap |
| **Appendices** | Tools & utilities table (Appendix A), distribution list (name + role) |
| **Generate PDF** | Risk summary overview + one-click generate & download |

### Finding Editor

The finding drawer (slides in from the right) contains:

- **From Template**: load a saved finding template from the library
- **Finding ID** (e.g. `INT-001`) and **Title**
- **Severity**: Critical / High / Medium / Low / Informational
- **Risk Rating Justification**: CVSS score and rationale
- **Description & Evidence**: the block editor (see below)
- **Impact**: combined technical and business impact narrative
- **Recommendations**: Markdown textarea
- **Affected Hosts**: add individual hostnames/IPs
- **References**: title + URL pairs (rendered as `**Title** - url` in the PDF)
- **Retest**: flag whether this finding requires retesting

### Content Block System

Findings and exploitation scenarios use a block-based editor where you can interleave three block types in any order:

| Block | What it does |
|-------|-------------|
| **Text** | Markdown narrative — supports `**bold**`, `*italic*`, `` `code` ``, lists, headings, fenced code blocks |
| **Code Block** | Syntax-highlighted code with a label, language selector, optional line highlighting, and a caption |
| **Screenshot** | Uploaded image with a drag-to-resize handle (10–100% width), a live PDF page preview, and a caption. Auto-assigned figure numbers in the PDF |

Between every block a `+` insert point appears on hover, letting you add any block type at any position. Blocks are reorderable via drag-and-drop (grip handle on the left).

In the PDF, content blocks render with sequential figure numbers across the entire report (scenarios first, then findings in section order).

### API Reference

All routes are prefixed `/api/`. Authentication uses `Authorization: Bearer <token>`.

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/auth/login` | OAuth2 password form → JWT (7-day) |
| `GET` | `/api/projects` | List all engagements |
| `POST` | `/api/projects` | Create engagement |
| `GET` | `/api/projects/{id}` | Get engagement with full report data |
| `PUT` | `/api/projects/{id}` | Update engagement |
| `DELETE` | `/api/projects/{id}` | Delete engagement |
| `POST` | `/api/projects/{id}/generate` | Generate PDF → streaming response |
| `POST` | `/api/projects/{id}/upload` | Upload screenshot → `{path, url}` |
| `GET` | `/api/media/{project_id}/{filename}` | Serve uploaded media *(no auth — needed for `<img src>` in editor)* |
| `GET/POST/PUT/DELETE` | `/api/clients` | Client CRUD |
| `GET/POST/PUT/DELETE` | `/api/domains` | Domain / infrastructure tracking |
| `GET/POST/PUT/DELETE` | `/api/designs` | Report design templates |
| `GET/POST/PUT/DELETE` | `/api/users` | User management *(admin only)* |
| `POST` | `/api/users/{id}/reset-password` | Reset user password *(admin only)* |

---

## Customisation

### Branding & Logo

The RootSec logo appears in two places:

1. **Running page header** (top-right of every page except the cover) sourced from `screenshots/logo.png`
2. **Cover page** (top-left, white/inverted version on dark background). Same file, CSS `filter: invert(1)` applied

To replace with your own logo:
```
screenshots/logo.png   ← replace with your firm logo (PNG, transparent background recommended)
```

The header logo is base64-encoded at render time by `generate.py` (`img_to_b64(BASE_DIR.parent / "screenshots" / "logo.png")`). No path configuration needed just replace the file.

### Report Templates (Jinja2)

Templates live in `pentest-report/templates/`. The master template `report.html.j2` includes all partials in section order. Each partial receives the full `report` dict as its context.

**Key template variables:**

| Variable | Description |
|----------|-------------|
| `report.meta` | Report ID, title, client, consultant, dates, classification |
| `report.introduction` | Preamble, objective, approach HTML, team, dates, caveats, authorisation HTML |
| `report.scope` | `scope.targets` dict (normalised from web format), `scope.roe_html`, `scope.methodology` |
| `report.executive_summary` | `narrative_html`, `key_findings[]`, `overall_risk_rating` |
| `report.technical_summary` | `narrative` (processed inline with `\| md` filter) |
| `report.enabled_sections` | Sorted, processed sections with enriched findings |
| `report.exploitation_scenarios` | Processed scenarios with `processed_content_blocks` or legacy fields |
| `report.risk_bar_svg` | Pre-rendered SVG string of the risk distribution chart |
| `report.tools_used` | List of `{tool, purpose, reference}` |
| `report._company_logo` | Base64 data URI of the firm logo |
| `has_tools` | Boolean — controls appendix lettering (A/B vs A/B/C) |

**Custom Jinja2 filters available in templates:**

| Filter | Description |
|--------|-------------|
| `\| md` | Convert Markdown string to HTML inline |
| `\| safe` | Mark HTML as safe (no escaping) |
| `\| fmt_date` | Format `YYYY-MM-DD` to `Month DD, YYYY` |
| `\| lower` | Lowercase (standard Jinja2) |

**Example — adding a field to a partial:**
```jinja2
{% if f.my_new_field %}
<div class="finding-section">
  <div class="finding-section-label">My New Field</div>
  <div class="finding-section-body md-content">{{ f.my_new_field | safe }}</div>
</div>
{% endif %}
```

### Styling (CSS)

All PDF styles live in `pentest-report/templates/assets/style.css`. The file is inlined into the HTML at render time (no external requests in the PDF).

**Key CSS variables:**
```css
:root {
  --text:       #1f2937;    /* primary body text */
  --text-sub:   #374151;    /* secondary text */
  --text-muted: #6b7280;    /* captions, labels */
  --navy:       #111827;    /* headings */
  --border:     #e5e7eb;    /* table borders, dividers */
  --border-mid: #d1d5db;
  --blue:       #1a7dd9;    /* accent — section underlines, links */
  --font-mono:  'Courier New', monospace;
}
```

**Page layout:**
- Paper size: A4 (`format: "A4"` in Playwright)
- Top margin: 88px (accommodates the running logo header)
- Bottom margin: 80px (accommodates the footer)
- Left/right margins: 0px (content padding is handled by CSS (72px on each side))

**Severity colour palette:**

| Severity | Colour |
|----------|--------|
| Critical | `#e63946` |
| High | `#f4802b` |
| Medium | `#f5c518` |
| Low | `#2196f3` |
| Informational | `#8b949e` |

### Adding a New Report Section

1. **Create the partial template:**
   ```
   pentest-report/templates/partials/_my_section.html.j2
   ```
   Give it a root `<div class="report-section page-break" id="my-section">`.

2. **Include it in `report.html.j2`** at the desired position:
   ```jinja2
   {% include 'partials/_my_section.html.j2' %}
   ```

3. **Process data in `generate.py`** inside `process_report()` if you need Markdown→HTML or other transforms:
   ```python
   my_section = report.setdefault("my_section", {})
   my_section["narrative_html"] = md_to_html(my_section.get("narrative", ""))
   ```

4. **Add to the TOC** in `_toc.html.j2`:
   ```jinja2
   <a class="toc-row toc-main toc-link" href="#my-section">
     <span class="toc-label">My Section</span>
     <span class="toc-leader"></span>
   </a>
   ```

5. **Add the UI** in `web/frontend/src/pages/ProjectDetail.tsx` under the appropriate tab (or add a new tab to the `TABS` array).

6. **Add the TypeScript type** in `web/frontend/src/types.ts` under `ReportData`.

---

## Data Schema Reference

### `ReportData` (top-level structure stored in the database)

```typescript
{
  meta: {
    report_id: string               // e.g. "RS-2026-001"
    title: string                   // e.g. "Penetration Test Report"
    subtitle: string
    client: {
      name: string
      contact_name: string
      contact_title: string
      contact_email: string
    }
    consultant: {
      firm: string
      name: string
      title: string
      email: string
      firm_logo_path: string        // path to logo (CLI only; web uses screenshots/logo.png)
    }
    dates: {
      assessment_start: string      // YYYY-MM-DD
      assessment_end: string
      report_date: string
      report_version: string
    }
    classification: string          // CONFIDENTIAL | RESTRICTED | INTERNAL | PUBLIC
    distribution: { name: string; role: string }[]
  }

  version_history: {
    version: string
    date: string                    // YYYY-MM-DD
    author: string
    changes: string
  }[]

  introduction: {
    preamble: string                // Markdown — no heading shown in PDF
    objective: string               // Markdown
    approach: string                // Markdown
    testing_team: { name: string; role: string }[]
    testing_dates: { start: string; end: string }
    caveats: string[]
    authorization: string           // Markdown
  }

  scope: {
    in_scope: { category: string; targets: string }[]   // comma-separated targets
    out_of_scope: string[]
    rules_of_engagement: string     // Markdown (stored, not currently rendered in PDF)
    methodology: string[]           // e.g. ["OWASP Testing Guide v4.2", "PTES"]
  }

  executive_summary: {
    overall_risk_rating: string     // Critical | High | Medium | Low | Informational
    narrative: string               // Markdown — intro paragraph
    key_findings: string[]          // Bullet points under Management Summary
  }

  technical_summary: {
    narrative: string               // Markdown — subsection under Executive Summary
  }

  risk_summary: {
    counts: {                       // auto-computed; override if needed
      critical?: number
      high?: number
      medium?: number
      low?: number
      informational?: number
    }
  }

  sections: {
    id: string
    title: string
    enabled: boolean
    findings: Finding[]
  }[]

  exploitation_scenarios: {
    id: string
    title: string
    severity: Severity
    finding_refs: string[]          // e.g. ["INT-001", "WEB-003"]
    narrative: string               // Markdown (legacy — use content_blocks instead)
    recommendation: string          // Markdown
    evidence: EvidenceItem[]        // legacy
    content_blocks?: ContentBlock[] // preferred — interleaved text/code/screenshot
    enabled: boolean
  }[]

  remediation_roadmap?: {
    timeframes?: {
      immediate?:   { items: RoadmapItem[] }
      short_term?:  { items: RoadmapItem[] }
      medium_term?: { items: RoadmapItem[] }
      long_term?:   { items: RoadmapItem[] }
    }
  }

  tools_used?: {
    tool: string
    purpose: string
    reference: string
  }[]
}
```

### `Finding`

```typescript
{
  id: string                        // e.g. "INT-001"
  title: string
  severity: 'critical' | 'high' | 'medium' | 'low' | 'informational'
  description: string               // Markdown (legacy — use content_blocks instead)
  risk_rating_justification: string // Markdown
  impact: {
    technical: string               // Markdown — the primary "Impact" field
    business: string                // Markdown — legacy; combined with technical in PDF
  }
  recommendations: {
    text?: string                   // Markdown (preferred single field)
    tactical?: string[]             // legacy array format
    strategic?: string[]            // legacy array format
  }
  affected_hosts: string[]
  references: { title: string; url: string }[]
  evidence: EvidenceItem[]          // legacy
  content_blocks?: ContentBlock[]   // preferred block format
  retest: { enabled: boolean }
}
```

### `ContentBlock`

```typescript
type ContentBlock =
  | { type: 'text'; content: string }
  | {
      type: 'code'
      label?: string
      language: string              // bash | python | powershell | sql | etc.
      content: string
      highlight_lines?: number[]
      caption?: string
    }
  | {
      type: 'screenshot'
      path: string                  // relative path under media/{project_id}/
      caption?: string
      width?: number                // percentage (10–100), default 100
    }
```

---

## Security Notes

- **Change the default credentials immediately**: `admin` / `admin123` are seeded on first startup
- **`SECRET_KEY` in `auth.py`**: hardcoded dev default. Set `SECRET_KEY` as an environment variable in production
- **`/api/media/` has no authentication**: intentional (browser `<img src>` tags cannot send Bearer tokens). Do not store sensitive non-image data in the media directory
- **SQLite**: fine for single-user or small team use. For multi-user production with concurrent writes, consider migrating to PostgreSQL via SQLAlchemy
- **Playwright/Chromium**: the PDF generator runs Chromium in headless mode. Run it inside the provided Docker container for an isolated, reproducible environment
