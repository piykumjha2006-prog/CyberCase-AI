# CyberCase AI — Build Plan & AI Prompts (3-Person Team)

Hackathon MVP · Web stack · Version 1.0

This doc gives you: (1) the tech stack per layer, (2) how to split the work between 3 people, (3) the **shared API contract** everyone must agree on before coding so no one blocks anyone else, and (4) ready-to-paste prompts for an AI coding assistant.

---

## 0. TL;DR — Stack at a glance

| Layer | Choice | Why |
|---|---|---|
| **Backend framework** | Python FastAPI + Uvicorn | Minimal boilerplate, auto Swagger docs, great for a 24-hour build |
| **Database** | SQLite via SQLModel (SQLAlchemy) | Zero setup, single file, perfect for hackathon. Swap to Postgres later |
| **File integrity** | `hashlib.sha256` on ingestion, server-side timestamp | Tamper-evident log with one line of code |
| **Extraction** | Python `re` (regex) + optional `spacy` NER | Deterministic, explainable — no hallucination risk |
| **Pattern detection** | Hand-written rule checks against extracted facts | Explainable under judge questioning, no black-box ML claim |
| **Report generation** | Single constrained LLM call (Anthropic API) | LLM only writes connective prose between pre-verified facts — never invents facts |
| **Frontend** | React + Vite + Tailwind | Fast to build, componentized, easy to demo on a projector |
| **File storage** | Filesystem (`/uploads`) + path in DB | Simplest option for a hackathon timeline |
| **Hosting for demo** | Local `uvicorn` + `vite dev`, optionally tunneled with `ngrok`/`cloudflared` | Only needed if judges access remotely; otherwise localhost is fine |

---

## 1. Work Split (3 members)

The seam is the **HTTP API**. Agree on the contract in Section 2 first, then each person builds against it independently — you only sync at the two integration points.

| | **Member A — Data Ingestion & Integrity** | **Member B — Logic Engine** | **Member C — Frontend & Demo** |
|---|---|---|---|
| **Owns** | Upload, hashing, structured extraction | Timeline, pattern detection, report generation | Dashboard UI, verification gate, integration |
| **Tasks** | File upload endpoint, SHA-256 hashing + timestamping, regex/NER field extraction, seed demo data | Timeline builder, 3 explainable pattern-detection rules, citation-constrained LLM report generation, verification gate logic | Upload panel, integrity log table, timeline view, flags panel, verification checkbox, report view, connection-status indicator |
| **Repo folders** | `backend/app/ingestion/` | `backend/app/logic/` | `frontend/` |
| **Primary skills** | Python, regex, file I/O | Python, rule design, LLM prompting | React, UI/UX |
| **Demo responsibility** | Upload + integrity log filling correctly | Timeline ordering + flags firing correctly, report generating | The live walkthrough — upload → timeline → flags → verify → report |

**Why this split:** ingestion is a clean, self-contained data-in problem; the logic engine consumes that data and is the technical heart of the "why this is trustworthy" story; the frontend is a pure consumer of both. All three can build in parallel against mocked JSON before the backend is fully wired.

---

## 2. Shared API Contract (agree on this FIRST)

All three members build against this. Freeze it before writing code. Base URL e.g. `http://localhost:8000`.

**`GET /api/health`** → `{ "status": "ok" }`

**`POST /api/upload`** — ingest one evidence file (multipart/form-data)
```
fields:  file (image/pdf/text)
returns: {
  "file_id": 7,
  "filename": "whatsapp_export.txt",
  "sha256_hash": "a1b2c3...",
  "uploaded_at": "2026-09-02T14:10:00Z",
  "extracted_facts": [
    { "fact_id": 1, "fact_type": "amount", "value": "25000", "source_snippet": "...transferred ₹25,000 to..." },
    { "fact_id": 2, "fact_type": "upi_id", "value": "scammer@upi", "source_snippet": "..." },
    { "fact_id": 3, "fact_type": "timestamp", "value": "2026-05-10T10:30:00Z", "source_snippet": "..." }
  ]
}
```

**`GET /api/integrity-log`** → array of every uploaded file's hash record
```json
[{ "file_id": 7, "filename": "whatsapp_export.txt", "sha256_hash": "a1b2c3...", "uploaded_at": "2026-09-02T14:10:00Z" }]
```

**`GET /api/timeline`** → chronological events built from all extracted facts
```json
[{ "event_id": 1, "timestamp": "2026-05-10T10:30:00Z", "description": "Suspicious WhatsApp message received", "file_id": 7, "source_snippet": "..." }]
```

**`GET /api/flags`** → triggered pattern-detection flags
```json
[{
  "flag_id": 1,
  "rule": "cross_case_identifier_match",
  "description": "UPI ID scammer@upi appeared in 3 separate evidence files",
  "related_file_ids": [7, 9, 12]
}]
```

**`POST /api/verify`** — human confirms review before report can generate
```
body:    { "reviewer_note": "optional string" }
returns: { "verified": true, "verified_at": "2026-09-02T15:00:00Z" }
```

**`POST /api/generate-report`** — only succeeds if `/api/verify` has been called
```
returns (200): {
  "report_markdown": "...",
  "citations": [{ "claim": "...", "file_id": 7 }],
  "generated_at": "..."
}
returns (403 if not verified): { "error": "Report generation requires human verification first" }
```

**Report generation rule (server-side):** the LLM call must receive only the structured facts already extracted — never raw file content — and its system prompt must instruct it to write connective narrative only, never introduce a fact not present in the input. Reject/flag any generated sentence that can't be mapped to a `file_id`.

**CORS:** backend must allow the frontend origin (`allow_origins=["*"]` is fine for the hackathon).

### Repo layout
```
cybercase-ai/
├── backend/
│   ├── app/
│   │   ├── ingestion/     # Member A
│   │   ├── logic/         # Member B
│   │   └── main.py        # shared FastAPI app, routers mounted here
│   └── uploads/           # saved evidence files
├── frontend/               # Member C — React + Vite dashboard
├── docs/api-contract.md
└── README.md
```

---

## 3. Prompts — Member A (Data Ingestion & Integrity)

Paste these one at a time into your AI coding assistant, in order.

### A1 — Scaffold FastAPI + SQLite
```
Create a Python FastAPI project called "cybercase-backend" using SQLModel over SQLite
(file cybercase.db). Set up uvicorn, CORS allowing all origins, and a /uploads static
mount for evidence files. Define two models: EvidenceFile (file_id pk, filename str,
sha256_hash str, uploaded_at datetime, file_path str) and ExtractedFact (fact_id pk,
file_id fk, fact_type str, value str, source_snippet str). Add GET /api/health returning
{"status":"ok"}. Auto-create tables on startup. Give me the full file tree and a README
with run commands.
```

### A2 — Upload + integrity hashing
```
Implement POST /api/upload accepting multipart/form-data with a single file field. On
each request: save the file to /uploads/{uuid}_{filename}, compute its SHA-256 hash,
record a server-side timestamp (do not trust file metadata), and insert a row into
EvidenceFile. Return the file_id, filename, sha256_hash, and uploaded_at. Write a pytest
test proving that two uploads of the same file content still get independent integrity
log entries with correct hashes.
```

### A3 — Structured extraction
```
Add a structured extraction function that runs on the text content of an uploaded file
(read as plain text; for now assume .txt exports, note where OCR would plug in for
images later). Using regex only (no LLM calls — this must be deterministic), extract:
monetary amounts (₹ figures), UPI IDs (pattern: alphanumeric@bankname), phone numbers
(Indian 10-digit format), and timestamps (common WhatsApp export formats like
"10/05/26, 10:30 AM -"). For each match, store a row in ExtractedFact with fact_type,
value, and a ~60-character source_snippet of surrounding text. Call this extraction
inside the /api/upload handler and include the extracted_facts array in the response,
matching this shape: [paste the extracted_facts array from the API contract].
```

### A4 — Integrity log endpoint
```
Add GET /api/integrity-log returning every EvidenceFile row as a JSON array matching:
[paste the integrity-log array shape from the contract]. Order by uploaded_at descending.
```

### A5 — Seed demo data
```
Write a seed script that creates 2 realistic sample scam case text files (a UPI fraud
case and a fake investment scam case) with fabricated but realistic WhatsApp-style
messages, timestamps, amounts, and UPI IDs — including one UPI ID that deliberately
repeats across both files so the pattern-detection rule has something to catch. Run
them through the upload + extraction pipeline so the database is pre-populated for
demoing without needing a live evidence set. Add a CLI flag to reset the DB and
/uploads folder between demo runs.
```

---

## 4. Prompts — Member B (Logic Engine)

Paste in order into your AI coding assistant.

### B1 — Timeline builder
```
Add GET /api/timeline to the FastAPI app. It should query all ExtractedFact rows where
fact_type = "timestamp", join back to their EvidenceFile, sort chronologically, and
return a JSON array matching: [paste the timeline array shape from the contract]. Each
event's description can be a short auto-generated label from the source_snippet (e.g.
truncate to ~60 chars). Write a pytest test with 3 out-of-order seeded facts proving
the endpoint returns them sorted correctly.
```

### B2 — Pattern detection engine
```
Add GET /api/flags implementing three explainable rules against the ExtractedFact table,
each returning a flag object matching this shape: [paste the flags array shape from the
contract]. Rules: (1) cross_case_identifier_match — the same value appears in fact_type
"upi_id" or "phone_number" across more than one file_id. (2) transaction_velocity — more
than 2 fact_type "amount" entries whose associated timestamp facts (same file_id) fall
within a 10-minute window. (3) scam_keyword_match — a source_snippet contains any phrase
from a seeded list: ["guaranteed returns", "urgent payment required", "share your OTP",
"double your money"]. Every returned flag must include the rule name and a human-readable
description referencing the actual matched values — never a generic "suspicious activity"
message. Write pytest tests proving each rule fires on crafted fixture data and does NOT
fire on clean data.
```

### B3 — Verification gate
```
Add POST /api/verify that accepts an optional reviewer_note string and records a
verification event (store verified=true and verified_at in a simple singleton or
per-case table — a single JSON row is fine for this hackathon's single-case scope).
Return {"verified": true, "verified_at": "..."}. This must exist and be checked by the
report generation endpoint before any report can be produced.
```

### B4 — Constrained LLM report generation
```
Add POST /api/generate-report. First check whether /api/verify has been called for this
case; if not, return a 403 with {"error": "Report generation requires human verification
first"}. If verified, gather the full timeline and flags data, and construct a prompt
for an LLM (Anthropic API, model claude-sonnet-4-6) that includes ONLY the structured
facts (never raw file content) and instructs it explicitly: "You are writing connective
narrative sentences between the facts provided below. Do not introduce any fact, name,
amount, or detail that is not present in the input. Every sentence you write must be
traceable to one of the provided fact IDs." Return the generated markdown report plus a
citations array mapping each claim back to a file_id, matching the contract shape. Add a
simple post-generation check that flags (logs a warning) if the LLM output contains a
number or identifier not present anywhere in the input facts.
```

### B5 — Hallucination guard test
```
Write a pytest test for /api/generate-report that feeds in a small fixed set of facts
(e.g. one amount, one UPI ID, one timestamp), calls the endpoint, and asserts that the
returned report_markdown contains only those values and no fabricated amount, name, or
identifier. This is the test we show if a judge asks "how do you know it doesn't
hallucinate."
```

---

## 5. Prompts — Member C (Frontend & Demo)

Paste in order into your AI coding assistant.

### C1 — Scaffold the dashboard
```
Create a React + Vite + Tailwind project called "cybercase-frontend". Set up 4
tabs/routes using react-router: "Upload" (default), "Timeline", "Flags", and "Report".
Add a top app bar with the app name "CyberCase AI" and a small connection-status dot
that pings GET {VITE_API_URL}/api/health every 10s (green = ok, red = down). Read the
backend base URL from an env var VITE_API_URL (default http://localhost:8000). Keep the
UI clean and high-contrast — this needs to demo reliably on a projector, not look fancy.
Give me the full file tree and all files.
```

### C2 — Upload panel + integrity log
```
On the "Upload" tab, add a file drop zone that POSTs each file to
{VITE_API_URL}/api/upload as multipart/form-data. Below it, show a table of the
integrity log fetched from GET /api/integrity-log with columns: filename, SHA-256 hash
(truncated to first/last 6 chars with a copy-full-hash button), and upload timestamp.
Refetch the integrity log after each successful upload. Show a toast on upload success
listing how many facts were extracted from that file.
```

### C3 — Timeline and flags views
```
Build the "Timeline" tab: fetch GET /api/timeline and render a vertical chronological
list, each event showing its timestamp, description, and a small tag linking back to
its source file_id. Build the "Flags" tab: fetch GET /api/flags and render each flag as
a card showing the rule name, the human-readable description, and the related file_ids
as tags. Poll both endpoints every 15s so newly uploaded evidence updates the views
without a manual refresh. Use mock JSON matching the API contract shapes for now so I
can build this before the backend is ready; make the mock data easy to swap out.
```

### C4 — Verification gate + report view
```
Build the "Report" tab: show a checkbox labeled "I have reviewed the timeline and
flagged evidence" that calls POST /api/verify when checked. Only after that call
succeeds, enable a "Generate Report" button that calls POST /api/generate-report and
renders the returned report_markdown (use a markdown renderer) with each citation
visibly linked to its file_id. If /api/generate-report returns a 403, show a clear
inline message telling the user to verify first — don't let the button be clickable
before verification succeeds.
```

### C5 — Demo polish
```
Add a loading skeleton for each tab while its data fetches, and a visible error state if
any API call fails (don't fail silently — show "Backend unreachable" with a retry
button). Add a small "Load demo case" button on the Upload tab that, when clicked, calls
a backend seed endpoint (or re-fetches pre-seeded data) so the whole flow can be
demonstrated instantly without live file uploads if something goes wrong on stage.
```

---

## 6. Integration Prompts (do together)

### I1 — Wire frontend to backend
```
Point the frontend VITE_API_URL at the running backend. Verify end to end: (1) health
dot turns green, (2) uploading the seeded UPI fraud case file populates the integrity
log and timeline, (3) uploading the second seeded file (with the repeated UPI ID)
triggers the cross_case_identifier_match flag on the Flags tab, (4) checking the
verification box enables Generate Report, (5) the generated report renders with correct
citations. Fix any CORS or field-name mismatches against the API contract.
```

### I2 — HTTPS tunnel (only if judges need remote access)
```
Give me the exact commands to expose the local backend and the Vite dev server over
HTTPS using cloudflared (or ngrok) so the app can be opened from another device if
needed for judging. Update VITE_API_URL to the tunneled backend URL.
```

### I3 — Demo dry-run checklist
```
Write a 1-page demo runbook: what to open, in what order, the exact narration for a
3-minute pitch, and a fallback plan if a live upload fails (use the "Load demo case"
button / pre-seeded data). Include the specific defensive lines to say if asked about
hallucination, legal admissibility, tampering, and accuracy — pull these from the PRD's
"Anticipated Judge Questions" section.
```

---

## 7. Suggested build order (parallel)

| Time | Member A | Member B | Member C |
|---|---|---|---|
| **Hour 0** | Agree API contract (Section 2), create repo | Agree API contract, create repo | Agree API contract, create repo |
| **Block 1** | A1 scaffold → A2 upload + hashing | Wait on A's schema, sketch B1/B2 logic against mock data | C1 scaffold → C2 upload panel (against mock API) |
| **Block 2** | A3 extraction → A4 integrity log endpoint | B1 timeline → B2 pattern detection (against A's real schema once ready) | C3 timeline + flags views (mock data) |
| **Integrate 1** | — | — | Swap frontend mocks for real A + B endpoints |
| **Block 3** | A5 seed demo data | B3 verify gate → B4 LLM report generation | C4 verification gate + report view |
| **Block 4** | Polish extraction edge cases | B5 hallucination guard test | C5 demo polish (loading/error states, demo-load button) |
| **Integrate 2** | I1 full end-to-end wiring, together | I1 full end-to-end wiring, together | I1 full end-to-end wiring, together |
| **Final** | I3 runbook prep | I3 runbook prep | I2 tunnel (if needed) + I3 runbook prep |

Key idea: get to a working end-to-end flow with **mocked/seeded data** as fast as possible — a complete, boring pipeline beats a brilliant extraction engine with no UI around it. Swap in real edge cases and polish only once everything connects.

---

## 8. What to have running for the judges

1. Upload tab: drop the seeded UPI fraud case file → integrity log fills with hash + timestamp.
2. Upload the second seeded file (shared UPI ID) → Timeline tab shows both events in order.
3. Flags tab: the cross-case identifier match flag appears, showing the exact matched UPI ID and both file_ids — not a vague "suspicious" label.
4. Report tab: verification checkbox is required before Generate Report is clickable.
5. Check verification → click Generate Report → markdown report renders with every claim visibly citation-linked to a file_id.
6. If asked "could this hallucinate," point to the constrained-prompt architecture and the B5 test proving it.

Sources: derived from your CyberCase AI PRD v2.0 (Sept 2, 2026) and the loophole-fixing discussion earlier in this conversation.