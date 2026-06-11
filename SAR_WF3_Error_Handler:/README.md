# SAR — Error Handler

> Workflow 3 of 3 · [Ver en español](./README.es.md)

Centralized error handling workflow for the SAR system. Automatically detects failures in any SAR workflow and sends an immediate email alert with the details needed to diagnose and fix the issue.

---

## Overview

This workflow acts as a safety net for the entire SAR system. When any node in **Workflow 1 (SAR)** or **Workflow 2 (SAR - Loop de Rechazo)** fails — whether due to a Google Sheets connection error, a Telegram API issue, or any other unexpected failure — this workflow triggers automatically and sends a structured alert email.

This is especially relevant in a production context where the system operates 24/7 without human supervision.

---

## How It Works

```
Error Trigger → Gmail (alert to admin)
```

| Step | Node | Description |
|------|------|-------------|
| 1 | Error Trigger | Activates automatically when any linked workflow fails |
| 2 | Send a message (Gmail) | Sends a structured alert email with error details |

---

## Alert Email Content

The email includes:

- **Workflow name** — which workflow failed (SAR or SAR - Loop de Rechazo)
- **Node name** — the specific node where the failure occurred
- **Error message** — the error description if available
- **Date and time** — when the failure happened
- **Direct link** — URL to the execution log in n8n for immediate debugging

---

## Setup

### 1. Import the workflow
Import `workflow-sar-error-handler.json` into your n8n instance.

### 2. Configure Gmail credentials
Connect your Gmail account in n8n and update the credential reference in the Gmail node.

### 3. Set the recipient email
Update the `sendTo` field in the Gmail node with the admin email address.

### 4. Link to other workflows
In **Workflow 1 (SAR)** and **Workflow 2 (SAR - Loop de Rechazo)**:
- Open workflow **Settings**
- Set **Error Workflow** → `SAR - Error Handler`
- Save and publish

### 5. Activate
Toggle the workflow to **Active**.

---

## Requirements

- n8n (self-hosted or cloud)
- Gmail OAuth2 credentials configured in n8n
- Workflow 1 and Workflow 2 must have this workflow set as their Error Workflow in Settings

---

## Part of the SAR System

| Workflow | Description |
|----------|-------------|
| [Workflow 1 — SAR](../wf1/) | Main flow: form intake, candidate filtering, Telegram notification |
| [Workflow 2 — SAR Loop de Rechazo](../wf2/) | Escalation loop: cycles through candidates until coverage is found |
| **Workflow 3 — SAR Error Handler** | **This workflow: centralized error detection and alerting** |

---

## Tech Stack

`n8n` `Gmail API` `Error Trigger` `JavaScript`

---

*Part of the SAR project — developed by [Fabricio Garrido](https://github.com/fabrogarrido) · [Ineo Digital Hub](https://iteradigitalhub.com)*
