# SAR — Workflow 1: Setup

> Part of the **Autonomous Replacement System (SAR)** — see [main README](../README.md)

## What does this workflow do?

Triggered when a professional submits the Google Form to request leave. It expands the date range, filters and sorts available candidates by priority, resets the candidate index to `-1` in Google Sheets, and immediately delegates to [WF2](../SAR_WF2_Loop_Rechazo/) via HTTP POST.

WF1 handles **no candidate contact**. All interaction logic — Telegram messages, response parsing, acceptance logging, notifications — lives exclusively in WF2.

## Node flow

| Node | Type | Function |
|------|------|---------|
| Google Sheets Trigger | Trigger | Detects a new row in the form responses tab |
| Expand Dates | Code JS | Calculates which shift dates fall within the requested range |
| Read Professionals | Google Sheets | Reads all professionals from the Professionals tab |
| Filter Candidates | Code JS | Filters by available day, ACTIVE status and excludes the requester. Sorts by priority |
| Split Dates | Code JS | Formats dates to DD/MM/YYYY and prepares the `pendingShifts` array |
| Reset Index | Google Sheets | Writes `-1` to the Status tab — WF2 increments to `0` on first call, landing on candidate 0 correctly |
| HTTP Request | HTTP | POST to the WF2 webhook with the full payload (dates, candidates, requester info) |

## Required configuration

### 1. Google Sheets
- Open the **Google Sheets Trigger** node and select your SAR spreadsheet
- Repeat for the **Read Professionals** and **Reset Index** nodes
- Verify that tab names match: `Form responses 1`, `Professionals`, `Status`

### 2. WF2 webhook URL
- Activate WF2 first to obtain its webhook URL
- In the **HTTP Request** node, paste the WF2 webhook URL

## Technical notes

- **Reset Index** writes `-1` (not `0`) so that WF2's first increment lands on candidate `0`. This is what allows WF2 to handle all candidates — including the first — without any special-casing in WF1.
- The HTTP Request body references `$('Split Fechas').first().json.*` explicitly, because **Reset Index** outputs the updated sheet row, not the candidate data. The expression reaches back to `Split Dates` to get the full payload.
- No Telegram, Gmail, or candidate-contact logic exists here. This workflow went from 20 nodes (v1) to 7 nodes (v2) by moving all that logic to WF2.
