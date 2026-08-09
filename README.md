# 🎫 AI Ticket Triage Checkpoint 01

An AI workflow that classifies support tickets, validates the model's output, routes risky/uncertain cases to a human, and never takes an autonomous customer-facing action.

---

## 📌 Overview

| | |
|---|---|
| **Type** | AI Workflow *(bounded LLM judgment, not an Agent)* |
| **Trigger** | Manual (v0/v1) → Webhook (planned) |
| **Model Provider** | OpenRouter (OpenAI-compatible API) |
| **Tool** | n8n |
| **Status** | v1 — Test Ready |

---

## 🧩 Six-Part Architecture

```
1. Trigger    →  Manual execution (v0/v1); webhook planned for v2
2. Context    →  { request_id, received_at, message }
3. Decision   →  LLM classifies category, confidence, risk (bounded, single decision)
4. Action     →  Auto-tag & route (internal only — no customer-facing action)
5. State      →  Full log entry per ticket, every path, no exceptions
6. Control    →  Human review gate; workflow pausable at platform level
```

---

## 🔁 Pipeline

```
Receive → Normalize → Duplicate Check → Classify → Validate → Route → Persist
```

```
01 Trigger
   ↓
02 Set fields  (request_id, received_at, message)
   ↓
02c Duplicate Check  → IF duplicate → Tag: duplicate ──┐
   ↓ (not duplicate)                                    │
02b Input Guard → IF invalid → Tag: error ──────────────┤
   ↓ (valid)                                             │
03 LLM (contract prompt, JSON-only)                       │
   ↓                                                      │
04 Parse + validate  → IF invalid → Tag: error ──────────┤
   ↓ (valid)                                              │
05 Route → Tag: human_review / Tag: auto_tagged ─────────┤
                                                           ↓
                                                     06 Persist
```

---

## 🧠 The Contract Prompt

Every LLM call uses the **JOB / CONTEXT / RULES / TOOLS / OUTPUT / CHECK** contract format:

```text
JOB:     Classify one support ticket into a routing decision.
CONTEXT: Free-text customer message. Allowed categories: billing, technical, account, spam, other.
RULES:   Return ONLY valid JSON. No markdown fences. No prose. Never invent a category.
TOOLS:   None. Classification only — no external action.
OUTPUT:  { category, confidence, summary, risk, needs_human }
CHECK:   Valid JSON · known category · confidence 0–1 · non-empty summary.
```

---

## 📦 Structured Output Schema

```json
{
  "category": "billing",
  "confidence": 0.92,
  "summary": "Duplicate charge reported",
  "risk": "medium",
  "needs_human": false
}
```

| Field | Type | Allowed values |
|---|---|---|
| `category` | string | `billing` \| `technical` \| `account` \| `spam` \| `other` |
| `confidence` | number | `0.0` – `1.0` |
| `summary` | string | non-empty, short |
| `risk` | string | `low` \| `medium` \| `high` |
| `needs_human` | boolean | — |

---

## ✅ Validation Rules

1. `category` must be in the allowed list
2. `confidence` must be a number between `0` and `1`
3. `summary` must be present and non-empty
4. `risk = "high"` → **forces** `needs_human = true` *(hard-coded override, never trusted to the model)*
5. `confidence < 0.6` → **forces** `needs_human = true`
6. Invalid/unparseable JSON → routed to the **error** path, never guessed

---

## 🚨 Failure Paths

| Failure | Handling |
|---|---|
| Empty / missing `message` | Caught by Input Guard before the LLM is ever called |
| Markdown-fenced JSON (` ```json `) | Stripped in code before parsing *(defense in depth)* |
| Invalid JSON from model | Routed to `error`, raw output preserved |
| Unknown category | Fails validation → `human_review` |
| Low confidence | Forced → `human_review` |
| High risk | Forced → `human_review`, regardless of confidence |
| API failure / timeout | `continueOnFail` + 15s timeout → logged as system error |
| Duplicate `request_id` | Detected via workflow static data → `duplicate_skipped`, LLM never called |

---

## 🔐 Human Approval Boundary

> The system **never** replies to a customer, refunds, cancels, or takes any irreversible action.
> Its only autonomous action is **internal tagging/routing**.
> Anything uncertain or risky is held at `human_review` — no exceptions, no overrides by the model itself.

---

## 🧪 Test Cases

| # | Input | Expected Result |
|---|---|---|
| **T1** | `I was charged twice this month for my subscription.` | `auto_tagged`, category=`billing` |
| **T2** | `it's not working, please help` | `human_review` *(low confidence)* |
| **T3** | `I'm contacting my lawyer about this fraudulent charge.` | `human_review` *(high risk override)* |
| **T4** | *(empty message)* | `error` — missing message |
| **T5** | Same `request_id` as T1, run again | `duplicate_skipped` — LLM never called |

---

## ⚙️ Setup

```bash
# 1. Import the workflow
n8n UI → Workflows → Import from File → checkpoint01_ticket_triage_v1_openrouter.json

# 2. Add credential
Settings → Credentials → New → Header Auth
  Header name:  Authorization
  Header value: Bearer <your_openrouter_api_key>

# 3. Attach credential to the "03 LLM" node

# 4. Confirm model slug in the request body
"model": "openai/gpt-4o-mini"   # or your confirmed OpenRouter slug

# 5. Run it
Click "Execute Workflow" on "01 Trigger"
```

---

## 📁 Files in This Project

```
checkpoint01_ticket_triage_v0_openrouter_happy_path.json   # straight-line happy path
checkpoint01_ticket_triage_v1_openrouter.json               # full error branching + duplicate check
capture_evidence.js                                          # Playwright evidence automation
README.md                                                     # this file
```

---

## ⚠️ Known Limitations (v1)

- **Duplicate check** uses n8n workflow **static data** — in-memory, not a real database. Fine for a demo, not production-grade.
- **Trigger** is manual only — webhook/form trigger planned for v2.
- **No customer-facing action** exists yet by design — this is intentional, not a gap.

---

## 🎯 Grading Alignment

| Dimension | Weight | Where it shows up |
|---|---|---|
| **Outcome** | 35% | Measurable, correctly-routed classification decisions |
| **Reliability** | 25% | Full error branching, fence-stripping, duplicate detection, no silent failures |
| **Clarity** | 20% | Six-part architecture, contract prompt, this README |
| **Safety** | 20% | Hard-coded human-review overrides, no autonomous external actions, synthetic data only |

---

## ✍️ Author

**Hafsa Anwar**
