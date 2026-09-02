# Product Requirements Document (PRD)

**Project Name:** CyberCase AI — Evidence Integrity & Investigation Timeline System
**Document Version:** 2.0 (Hardened)
**Author:** Piyush
**Status:** Draft — Hackathon MVP Scope
**Event:** Recursion 2.0, VIT Chennai (24-hour hackathon)

---

## 1. Overview

CyberCase AI is an evidence-organization and integrity tool for digital scam victims and investigators. It ingests scattered digital evidence (chats, emails, screenshots, transaction records, call recordings), structurally extracts facts from it, builds a tamper-logged chronological timeline, flags explainable suspicious patterns, and — only after human verification — generates a citation-linked investigation summary.

**It is explicitly not** an autonomous system that writes legal documents or "decides" anything. Every fact in its output traces back to a specific, hashed source file. The AI's only generative role is writing connective prose around facts a human has reviewed.

**One-line pitch:**
"We turn a scam victim's scattered evidence into a court-ready, tamper-logged, human-verified timeline in minutes instead of the hours investigators currently spend doing it by hand."

---

## 2. Problem Statement

Scam victims accumulate evidence across WhatsApp, email, bank SMS, and call logs with no structure. Investigators currently spend hours manually reconstructing a timeline before analysis can even begin — during which momentum, and often recoverable money, is lost.

Existing "AI report generator" approaches for this problem carry real risk: unverified AI output could hallucinate evidence, lack any chain-of-custody, and be legally indefensible. CyberCase AI is designed from the ground up to avoid all three.

---

## 3. Goals & Non-Goals

### Goals
- Ingest multi-source digital evidence (chat exports, screenshots, transaction SMS, emails, call recordings)
- Hash and timestamp every file on ingestion to create a tamper-evident integrity log
- Extract structured facts (not free-form AI reading) from each evidence item
- Auto-build a chronological, source-linked timeline from extracted facts
- Flag suspicious patterns using explainable, rule-based logic — not opaque ML claims
- Require explicit human verification before any report is finalized
- Generate a citation-linked investigation summary suitable for handing to police/investigators

### Non-Goals (explicitly out of scope — say this if asked)
- This is **not** a legal document generator or FIR replacement
- This is **not** an autonomous decision-maker — a human always signs off
- This does **not** claim forensic-grade chain-of-custody — the integrity log is a lightweight, demoable analog of that concept, not a certified forensic tool
- This does **not** claim a trained-model accuracy metric — pattern detection is rule-based and explainable by design

---

## 4. Positioning (why this framing matters)

| Weak framing (avoid) | Hardened framing (use) |
|---|---|
| "AI agent that collects and analyzes evidence" | "Structured pipeline that extracts facts mechanically, organizes them, and only summarizes what's already verified" |
| "Police-ready report" | "Investigator's evidence-organization tool that produces a police-ready *summary and index*, reviewed and signed off by a human" |
| "AI detects suspicious patterns" | "Explainable rule-based flags — each one shows the exact rule that triggered it" |
| "We know this works" (accuracy claims) | Live before/after demo — the visual contrast is the proof, not a number |

---

## 5. Architecture

### 5.1 Two-Stage Pipeline (hallucination-proofed by design)

**Stage 1 — Structured Extraction Layer**
Deterministic, not generative. Rules + NER/regex pull concrete fields from raw evidence:
- Sender/recipient identifiers
- Timestamps
- Monetary amounts
- Account numbers / UPI IDs
- Phone numbers
- Scam-indicator keywords

Nothing at this stage can be "invented" — it's pattern extraction, not language generation.

**Stage 2 — Organization Layer**
An LLM arranges and summarizes the *already-extracted* structured facts into readable prose. Every sentence in the final output is citation-linked back to a specific source item (file ID, timestamp, hash).

> Core defensive line for judges: **"Our AI never invents evidence — it only organizes and cites evidence that's already been structurally extracted."**

### 5.2 Evidence Integrity Log

Every uploaded file is SHA-256 hashed and server-side timestamped on ingestion, independent of the file's own (spoofable) metadata. This produces a visible "Integrity Log" table in the final report — a lightweight, demoable stand-in for chain-of-custody that most competing teams won't have considered.

### 5.3 Pattern Detection (rule-based, explainable)

| Rule | What it flags |
|---|---|
| Cross-case identifier match | Same UPI ID / account number / phone number appears across multiple evidence items or cases |
| Transaction velocity | Multiple transfers within a short time window |
| Keyword match | Message content matches a seeded list of known scam phrasing |

Each flag displays the exact rule that fired — no black-box ML claims that can't be defended under questioning.

### 5.4 Human-in-the-Loop Checkpoint

A "Human-Verified ✓" checkbox gates report generation. The system cannot produce a final report without an explicit human confirmation step. This directly preempts the "AI replacing police/judgment" objection.

---

## 6. Core Features (MVP scope for 24 hours)

1. **Evidence Upload** — multi-file upload (images, text exports, PDFs)
2. **Integrity Hashing** — SHA-256 hash + server timestamp per file, on ingestion
3. **Structured Extraction** — regex/NER pipeline pulling amounts, IDs, timestamps, keywords
4. **Auto-Built Timeline** — chronological view, each event linked to its source file + hash
5. **Explainable Pattern Flags** — hardcoded rule checks against extracted fields, shown with trigger reason
6. **Human Verification Gate** — checkbox required before report generation unlocks
7. **Citation-Linked Report Generation** — template-based document; LLM only writes connective narrative between cited, pre-verified facts

---

## 7. Tech Stack (buildable in 24h)

- **Backend:** FastAPI
- **Extraction:** Python regex + a lightweight NER model (or rule-based field matching if time-constrained)
- **Hashing:** Python `hashlib.sha256`, one line per uploaded file
- **Storage:** SQLite/Postgres for structured facts + file metadata
- **Report generation:** Template engine for structure + single LLM call for connective prose, strictly constrained to cite provided facts
- **Frontend:** Simple dashboard — timeline view, integrity log table, flagged patterns list, verification checkbox, "Generate Report" action

---

## 8. Demo Script (this is what needs to survive Q&A)

1. **Before** — show a messy folder: WhatsApp exports, screenshots, bank SMS.
2. **Upload** — show structured extraction happening; name the actual fields being pulled, not a vague loading spinner.
3. **Timeline** — auto-built, each event visibly linked to its source file + hash.
4. **Flag** — show one pattern flagged with its explainable rule (e.g., "same UPI ID appeared in 3 separate reports").
5. **Checkpoint** — show the Human-Verified checkbox being checked before report generation activates.
6. **Report** — generate live; every claim in it visibly links back to source evidence.

---

## 9. Anticipated Judge Questions & Prepared Answers

| Question | Answer |
|---|---|
| "Could the AI hallucinate evidence?" | "No — extraction is rule-based and deterministic. The LLM only organizes facts that were already mechanically pulled out; it never reads raw evidence and freely writes conclusions." |
| "Is this legally admissible?" | "We don't claim that. This produces an investigator's evidence index — a human reviews and verifies before anything is finalized." |
| "How do you prevent tampered/faked evidence?" | "Every file is hashed and timestamped on ingestion, independent of its own metadata — visible in the Integrity Log." |
| "Is this just a ChatGPT wrapper?" | "No — pattern detection is rule-based and explainable, shown with the exact triggering condition. The LLM's only job is prose, not detection or decision-making." |
| "What's your accuracy?" | We don't claim an unverifiable number — the live before/after demo is the proof. |

---

## 10. What NOT to Say on Stage

- Don't claim an accuracy percentage.
- Don't call it a "police report" — call it an "investigation summary" or "evidence index."
- Don't say "AI agent" without immediately clarifying the extraction/organization split.

---

## 11. Success Criteria for Demo

- Timeline builds correctly from at least 2 seeded evidence sets (e.g., a UPI fraud case, a fake investment scam case)
- At least one pattern flag fires correctly and visibly shows its trigger rule
- Integrity log displays hash + timestamp for every uploaded file
- Report generation is blocked until the Human-Verified checkbox is checked
- Every claim in the generated report is traceable to a source file

---

## 12. Build Priority Order (if time runs short, cut from the bottom up)

1. Upload + hashing + timeline (core, non-negotiable)
2. Structured extraction for at least amounts, timestamps, and one identifier type
3. Human-verification checkbox gate
4. One working pattern-flag rule
5. Citation-linked report generation
6. Additional pattern rules / polish