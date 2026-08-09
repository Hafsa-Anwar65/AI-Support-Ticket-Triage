# 🧪 Test Cases

**Deliverable 4: Test & Break**

This document covers the five test cases used to validate the v1 workflow (`checkpoint01_ticket_triage_v1_openrouter.json`), the expected vs. actual behavior for each, and the evidence checklist.

---

## 🎯 Purpose

> Build the happy path — then immediately break it.

Each test case targets a **different failure mode** defined in the six-part architecture's Validate and Route steps. Together they prove the system doesn't just work when everything goes right — it fails safely and observably when it doesn't.

---

## 📋 Test Case Table

| # | Name | `request_id` | `message` | Expected `status` in `06 Persist` |
|---|---|---|---|---|
| **T1** | Happy path | `REQ-T1` | `I was charged twice this month for my subscription.` | `auto_tagged`, `category: billing` |
| **T2** | Low confidence | `REQ-T2` | `it's not working, please help` | `human_review` *(confidence < 0.6 override)* |
| **T3** | High risk | `REQ-T3` | `I'm contacting my lawyer about this fraudulent charge.` | `human_review` *(risk = high override)* |
| **T4** | Invalid input | `REQ-T4` | *(leave empty — clear the field entirely)* | `error`, `error: "missing or empty message field"` |
| **T5** | Duplicate | `REQ-T1` *(reused)* | Same as T1 - run **after** T1 | `duplicate_skipped`, LLM never called |

---

## ▶️ How to Run Each Test

```
1. Open "02 Set fields"
2. Edit request_id and message to match the row above
3. Click "Execute Workflow"
4. Open "06 Persist" → copy or screenshot the output JSON
```

> ⚠️ **T5 depends on T1.** Run T1 first so its `request_id` exists in memory, *then* run T5 with the same ID.

---

## 🧾 What Each Test Actually Proves

```
T1  →  The full happy path works end-to-end, clean classification
T2  →  The system doesn't guess when the model itself is unsure
T3  →  Safety override works even when the model is confident
T4  →  Bad input never reaches the LLM — caught by Input Guard
T5  →  The same ticket is never processed twice
```

---

## 📸 Evidence Checklist

- [ ] T1 — screenshot / output JSON saved
- [ ] T2 — screenshot / output JSON saved
- [ ] T3 — screenshot / output JSON saved
- [ ] T4 — screenshot / output JSON saved
- [ ] T5 — screenshot / output JSON saved

**Naming convention for saved evidence:**
```
evidence/T1_output.png
evidence/T2_output.png
evidence/T3_output.png
evidence/T4_output.png
evidence/T5_output.png
```

*(Matches the output of `capture_evidence.js` if using the Playwright automation script.)*

---

## 🔎 Real Issues Found During Testing

Two genuine failures were discovered and fixed while running these tests — kept here as part of the evidence trail, not hidden:

| Issue | Cause | Fix |
|---|---|---|
| `Invalid JSON from model` on a clean happy-path input | Model wrapped valid JSON in ` ```json ` markdown fences | Fence-stripping added to `04 Parse + validate` before `JSON.parse()` |
| `raw_output` fields showing `null` in `06 Persist` | n8n's Set node "Include Other Input Fields" was off by default on the Tag nodes, silently dropping fields | Enabled `includeOtherFields: true` on `Tag: error`, `Tag: human_review`, `Tag: auto_tagged` |

---

## ✅ Result Summary *(fill in after running)*

| # | Actual `status` | Matched Expected? | Notes |
|---|---|---|---|
| T1 | | ☐ Yes ☐ No | |
| T2 | | ☐ Yes ☐ No | |
| T3 | | ☐ Yes ☐ No | |
| T4 | | ☐ Yes ☐ No | |
| T5 | | ☐ Yes ☐ No | |

---

## ✍️ Author

**Hafsa Anwar**
