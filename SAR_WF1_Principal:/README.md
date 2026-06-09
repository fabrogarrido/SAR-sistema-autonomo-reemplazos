# SAR — Workflow 1: Main Flow

> Part of the **Autonomous Replacement System (SAR)** — see [main README](../README.md)

## What does this workflow do?

Triggered when a professional submits the Google Form to request leave. It processes the date range, identifies the highest-priority available candidate, and sends them an interactive query via Telegram. If the candidate rejects or partially accepts, it delegates to [WF2](../SAR_WF2_Loop_Rechazo/) to continue the escalation.

## Node flow

| Node | Type | Function |
|------|------|---------|
| Google Sheets Trigger | Trigger | Detects a new row in the form responses tab |
| Expand Dates | Code JS | Calculates which shift dates fall within the requested range |
| Read Professionals | Google Sheets | Reads all professionals from the Professionals tab |
| Filter Candidates | Code JS | Filters by available day, ACTIVE status and excludes the requester. Sorts by priority |
| Split Dates | Code JS | Formats dates to DD/MM/YYYY and prepares the `pendingShifts` array |
| Reset Index | Google Sheets | Writes `0` to the Status tab to start from the highest-priority candidate |
| Ask and Wait | Telegram | Sends a Custom Form with dynamic date checkboxes to the candidate |
| Process Response | Code JS | Calculates `acceptedDates` and `remainingDates` based on the selection |
| IF (canCover) | IF | Evaluates whether the professional accepted at least one date |
| Replacement Confirmed | Telegram | Confirms to the professional which dates they will cover |
| Find Requester | Google Sheets | Looks up the Telegram ID of the requesting professional |
| You Are Covered | Telegram | Notifies the requester of the coverage status |
| Notify Management | Gmail | Sends a summary to management and administration |
| Expand Accepted Dates | Code JS | Generates one item per accepted date |
| Log Replacement | Google Sheets | Writes one row per date to the Replacements tab |
| IF Any Dates Left? | IF | Checks if `remainingDates.length > 0` |
| All Covered | Telegram | Final message to the requester when all dates are covered |
| Prepare Escalation | Code JS | Prepares the payload with remaining dates and candidates |
| HTTP Request | HTTP | POST to the WF2 webhook to continue escalation |

## Required configuration

### 1. Google Sheets
- Open the **Google Sheets Trigger** node and select your SAR spreadsheet
- Repeat for the **Read Professionals**, **Reset Index**, **Find Requester** and **Log Replacement** nodes
- Verify that tab names match: `Form responses 1`, `Professionals`, `Status`, `Replacements`

### 2. Telegram
- In n8n → Credentials → create a **Telegram API** credential with your bot token
- Assign it to all Telegram nodes in the workflow

### 3. Gmail
- In n8n → Credentials → connect your Gmail account via OAuth2
- In the **Notify Management** node, replace `your-email@gmail.com` with the real management and admin emails

### 4. WF2 webhook URL
- In the **HTTP Request** node, replace `https://YOUR-N8N-DOMAIN.com/webhook/sar-rejection` with the real WF2 webhook URL

## Technical notes

- The **Filter Candidates** node automatically excludes the requester by comparing `p.NAME === request.requester`
- **Reset Index** at the start guarantees every new process always begins from the priority 1 candidate
- The timeout on the **Ask and Wait** node is set to 6 hours; if the professional doesn't respond, the execution stops
- Custom Form checkboxes are generated dynamically using `.map()` over the `pendingShifts` array
