# SAR — Autonomous Replacement System

**End-to-end automation system in production** for the Mental Health Department of a public hospital in Buenos Aires, Argentina.

Replaces a critical process that relied on phone calls and WhatsApp messages with an algorithmic, fully autonomous system available 24/7 — at zero infrastructure cost.

> **Status:** v1 in production · Infrastructure cost: $0

---

## The problem it solves

When a psychologist couldn't cover their shift, the process looked like this:

1. The professional notifies they can't attend
2. The department head starts calling colleagues one by one until someone picks up
3. Hours can pass without confirmed coverage
4. Agreements are made in private chats, with no traceable record
5. The same people always get called — generating ongoing distribution conflicts

**In a mental health department, an uncovered shift has real consequences for patients.**

---

## The solution

SAR acts as an impartial coordinator that operates without human intervention:

```
Professional submits Google Form
          ↓
SAR calculates which shift dates are left uncovered
          ↓
Contacts available candidates via Telegram (in priority order)
          ↓
Professional selects dates using interactive checkboxes
          ↓
If rejected or partially accepted → escalates to the next candidate
          ↓
Logs each coverage to Google Sheets + notifies management via email
```

---

## Stack

| Component | Tool |
|---|---|
| Automation engine | n8n (self-hosted on VPS with Docker) |
| Database | Google Sheets |
| Entry form | Google Forms |
| Communication channel | Telegram Bot API |
| Admin notifications | Gmail API |
| Language | JavaScript (ES6+) |
| Infrastructure | Contabo VPS · Docker |

---

## Architecture

The system runs across three independent workflows. Two handle the core replacement logic, and one acts as a centralized error handler.

```
        Google Forms
              │
              ▼
┌─────────────────────────────────┐
│    Workflow 1 — Main flow       │
│                                 │
│  · Date range expansion         │
│  · Candidate filtering and      │
│    priority ordering            │
│  · Dynamic custom form          │
│    with checkboxes (Telegram)   │
│  · Acceptance logging           │
│  · Multi-channel notifications  │
└──────────────┬──────────────────┘
               │
               │ HTTP POST (rejection / partial acceptance)
               ▼
┌─────────────────────────────────┐
│   Workflow 2 — Rejection loop   │
│                                 │
│  · Reads current candidate      │
│    index (Google Sheets)        │
│  · Calculates next candidate    │
│  · Repeats contact flow         │
│  · Calls itself via HTTP POST   │
│    (stateless loop)             │
└─────────────────────────────────┘

        ↕ on any failure in WF1 or WF2

┌─────────────────────────────────┐
│   Workflow 3 — Error Handler    │
│                                 │
│  · Triggered automatically on   │
│    any workflow failure         │
│  · Sends alert email with       │
│    workflow, node, error and    │
│    direct link to execution log │
└─────────────────────────────────┘
```

### Why two separate workflows for the core logic?

n8n is stateless by design: each webhook execution is independent and shares no memory with previous runs. A single workflow with multiple entry points would execute all its nodes once per incoming connection, generating duplicate records.

**The solution:** splitting the flows guarantees a single clean entry point per workflow. The current candidate index is persisted in a `Status` tab in Google Sheets, acting as an *external state store* — the same conceptual pattern as Redis or DynamoDB at larger scale, implemented at zero cost.

---

## Technical decisions

| Decision | Discarded alternative | Why |
|---|---|---|
| Two separate workflows | Loop within a single WF | Prevents double execution of nodes with multiple entry points |
| Google Sheets as state store | `$getWorkflowStaticData` | n8n's internal variable doesn't persist across separate webhook executions |
| Dynamic checkboxes via Custom Form | Accept / Reject buttons | Allows selection of multiple dates in a single interaction |
| Dropdown in Google Forms | Free text field | Ensures the name matches exactly with the record in the Sheet |
| Dedicated error handler workflow | Individual try/catch per node | Centralized alerting with zero changes to core logic |

---

## Features — v1 vs v2

| Feature | v1 | v2 |
|---|---|---|
| Automatic leave detection via Forms | ✅ | |
| Filtering by availability and priority | ✅ | |
| Requester excluded from candidate list | ✅ | |
| Multi-date selection with checkboxes | ✅ | |
| Automatic escalation to next candidate | ✅ | |
| Real-time notifications (Telegram + Gmail) | ✅ | |
| Automatic logging to Google Sheets | ✅ | |
| Management alert when no coverage found | ✅ | |
| 6-hour inactivity timeout | ✅ | |
| Centralized error handling + email alerts | ✅ | |
| Automatic retry after timeout | | 📋 |
| Rule: last to cover goes to end of list | | 📋 |
| Coverage limit per person per period | | 📋 |
| Inline Telegram buttons without external link | | 📋 |

---

## Repository structure

```
SAR/
├── README.md                             ← this file (English)
├── README.es.md                          ← Spanish version
├── SAR_WF1_Principal/
│   ├── SAR_WF1_Principal.json            ← n8n exported workflow
│   ├── README.md                         ← WF1 documentation (English)
│   └── README.es.md                      ← WF1 documentation (Spanish)
├── SAR_WF2_Loop_Rechazo/
│   ├── SAR_WF2_Loop_Rechazo.json         ← n8n exported workflow
│   ├── README.md                         ← WF2 documentation (English)
│   └── README.es.md                      ← WF2 documentation (Spanish)
└── SAR_WF3_Error_Handler/
    ├── workflow-sar-error-handler.json   ← n8n exported workflow
    ├── README.md                         ← WF3 documentation (English)
    └── README.es.md                      ← WF3 documentation (Spanish)
```

---

## Installation

### Requirements

- n8n installed (self-hosted or cloud)
- Google Sheets with the structure described below
- Telegram bot created with @BotFather
- Gmail account connected to n8n via OAuth2
- Google Forms linked to the `Form responses 1` tab

### Steps

1. **Import the workflows into n8n**
   Menu → Workflows → Import from file → import WF1, WF2 and WF3

2. **Configure credentials in n8n**
   - Google Sheets: OAuth2
   - Telegram: bot token
   - Gmail: OAuth2

3. **Update references in both workflows**
   - WF2 webhook URL in the HTTP Request nodes
   - Destination email in the Gmail nodes
   - Spreadsheet ID in the Google Sheets nodes

4. **Link the error handler**
   In WF1 and WF2 Settings → Error Workflow → select `SAR - Error Handler`

5. **Activate WF2 and WF3 first**, then activate WF1

6. **Load the professionals** into the `Professionals` tab in the Sheet

7. **Test** by submitting a sample response from Google Forms

### Google Sheets structure

**Tab: Professionals**

| PRIORITY | NAME | CATEGORY | STATUS | AVAILABILITY | TELEGRAM ID | NOTES |
|---|---|---|---|---|---|---|
| 1–15 | Name | Staff / G / C / R | ACTIVE | MON, TUE... | Numeric ID | Free text |

**Tab: Replacements** (auto-generated)

| DATE | DAY | COVERED_BY | REQUESTER | REASON |
|---|---|---|---|---|
| DD/MM/YYYY | TUESDAY | Professional X | Professional Y | Vacation |

**Tab: Status** (system persistent memory)

| KEY | VALUE |
|---|---|
| candidateIndex | 0 |

---

## Author

Built by **Fabricio Garrido** — [LinkedIn](https://linkedin.com/in/fabriciogarrido) · [GitHub](https://github.com/fabrogarrido)
