# SAR — Workflow 2: Rejection Loop

> Part of the **Autonomous Replacement System (SAR)** — see [main README](../README.md)

## What does this workflow do?

Handles **all candidate contact logic** — from the very first candidate to the last. Triggered by an HTTP Request from WF1 on every new replacement request (not just on rejection). It advances through the priority list, sending an interactive Telegram form to each candidate, until all dates are covered or all available candidates are exhausted.

WF2 is the single source of truth for: candidate contact, response parsing, acceptance logging, requester notification, and management email. In v1, this logic was duplicated between WF1 and WF2. In v2, it lives here only.

**How the index works:** WF1 resets the Status tab to `-1`. On the first call, WF2 reads `-1`, increments to `0`, and contacts candidate `0` (the highest-priority one). On each subsequent rejection, WF2 calls itself and increments by one again.

## Why a separate workflow?

n8n is **stateless by design**: each webhook execution is independent and shares no memory with previous runs. If the rejection loop were inside WF1, a node with multiple incoming connections would execute twice — once per connection — generating duplicate and inconsistent data.

Splitting into two workflows guarantees that each one has **a single clean entry point**, eliminating the double-execution problem. The current candidate index is persisted in the `Status` tab of Google Sheets, acting as an **external state store** across independent executions.

## Node flow

| Node | Type | Function |
|------|------|---------|
| Webhook | Webhook | Receives the payload with pending dates and candidates from WF1 or itself |
| Read Index | Google Sheets | Reads `candidateIndex` from the Status tab |
| Next Candidate | Code JS | Calculates the next candidate and prepares `pendingShifts` |
| IF1 (noCandidates) | IF | Checks whether any candidates are still available |
| Notify Management — No Coverage | Gmail | Alert email when all candidates have been exhausted |
| Save Index | Google Sheets | Writes the new index to the Status tab |
| Ask and Wait 2 | Telegram | Sends a Custom Form with dynamic checkboxes to the next candidate |
| Process Response 2 | Code JS | Calculates `acceptedDates` and `remainingDates` |
| IF (canCover) | IF | Evaluates whether the professional accepted at least one date |
| Replacement Confirmed 2 | Telegram | Confirms to the professional which dates they will cover |
| Find Requester 2 | Google Sheets | Looks up the Telegram ID of the requester |
| You Are Covered 2 | Telegram | Notifies the requester of the updated coverage status |
| Notify Management 2 | Gmail | Email to management and administration with the summary |
| Expand Accepted Dates 2 | Code JS | Generates one item per accepted date |
| Log Replacement 2 | Google Sheets | Writes one row per date to the Replacements tab |
| IF Any Dates Left? 2 | IF | Checks if `remainingDates.length > 0` |
| All Covered 2 | Telegram | Final message to the requester when all dates are covered |
| Prepare Escalation 2 | Code JS | Prepares the payload for the next candidate |
| HTTP Request 2 | HTTP | POST to the same webhook — restarts the loop with the next candidate |

## Required configuration

### 1. Webhook
- Activate this workflow first to obtain the webhook URL
- Copy that URL and paste it into:
  - The **HTTP Request** node in WF1 (initial delegation)
  - The **HTTP Request** and **HTTP Request 2** nodes in this workflow (self-loop on rejection)
- Replace all instances of `https://YOUR-N8N-DOMAIN.com/webhook/sar-rejection`

### 2. Google Sheets
- Assign your SAR spreadsheet to the following nodes: **Read Index**, **Save Index**, **Find Requester 2**, **Log Replacement 2**
- Verify that tab names match: `Status`, `Professionals`, `Replacements`

### 3. Telegram
- Use the same **Telegram API** credential configured in WF1
- Assign it to all Telegram nodes in this workflow

### 4. Gmail
- Use the same Gmail credential configured in WF1
- In the **Notify Management — No Coverage** and **Notify Management 2** nodes, replace `your-email@gmail.com` with the real emails

## Technical notes

- The **Next Candidate** node accesses the webhook body with `JSON.parse(rawBody[''])` — n8n wraps the body in an empty key when Content-Type is `application/json`; the code handles both cases with a try/catch
- The loop is self-limiting: when `candidateIndex >= candidates.length`, IF1 routes to the management notification node and the flow ends cleanly
- The index is incremented **before** sending the message, so the value in the Status tab always reflects the candidate currently being contacted
- On the **first call from WF1**, the Status tab contains `-1`. `Next Candidate` reads it, increments to `0`, and contacts `candidates[0]` — the highest-priority professional
- 6-hour timeout on **Ask and Wait 2**; if the professional doesn't respond, that execution stops (pending: automatic retry with the next candidate)
