# New User Account Provisioning Pipeline

An n8n automation that turns a new row in a Google Sheet into a fully provisioned (and logged, and announced) new hire — no manual IT/HR steps required.

---

## Table of Contents

1. [Summary](#summary)
2. [Why This Exists](#why-this-exists)
3. [Who Uses It](#who-uses-it)
4. [Architecture Overview](#architecture-overview)
5. [Data Schema (Source Sheet)](#data-schema-source-sheet)
6. [Prerequisites & Tech Stack](#prerequisites--tech-stack)
7. [Setup & Installation](#setup--installation)
8. [Main Workflow — Node by Node](#main-workflow--node-by-node)
9. [Sub-Workflows — Mock Provisioning Endpoints](#sub-workflows--mock-provisioning-endpoints)
10. [Error Handling Design](#error-handling-design)
11. [Airtable Log Schema](#airtable-log-schema)
12. [Notifications](#notifications)
13. [Security Notes](#security-notes)
14. [Testing the Pipeline](#testing-the-pipeline)
15. [Troubleshooting](#troubleshooting)
16. [Known Limitations](#known-limitations)
17. [Roadmap / Future Improvements](#roadmap--future-improvements)
18. [FAQ](#faq)

---

## Summary

This n8n workflow watches a Google Sheet of new hires and, whenever a row is added or updated, normalizes the employee's data, routes them into one of four department branches (Engineering, Sales, Marketing, Human Resources), and calls that department's provisioning endpoint to simulate account creation. All branches are merged back together, logged to Airtable (upserted by Employee ID to avoid duplicates), and followed by a Slack message and email announcing the new hire. If any step along the way fails — a provisioning call or the Airtable write — a centralized error handler posts a detailed failure alert to Slack instead of failing silently.

The four provisioning endpoints are currently **mock sub-workflows** (simple Webhook → Respond to Webhook pairs) standing in for real external APIs like Google Workspace Admin, Slack, GitHub, and Jira — they can be swapped for real API calls later without changing the rest of the pipeline's structure.

## Why This Exists

Manual new-hire account provisioning is repetitive, easy to forget steps in, and slow — especially once a company has several departments, each needing a different set of tools (Engineering needs GitHub/Jira/AWS, Sales needs a CRM and dialer, etc.). Without automation:

- HR/IT has to remember to create accounts across every relevant tool, one by one, per hire.
- There's no single source of truth for "has this person been provisioned yet, and with what?"
- Failures (a tool's API being down, a typo in an email) can go unnoticed until the new hire shows up without access.
- Managers and IT don't have a reliable, consistent way of knowing provisioning is done.

**Benefits to the company:**

- **Saves IT/HR time** — account creation across multiple systems happens automatically instead of someone manually clicking through each admin console per new hire.
- **Faster time-to-productivity** — a new hire's accounts can be ready before their first day instead of hours or days into it.
- **Consistency** — every new hire goes through the exact same steps for their department, so nobody accidentally skips a tool.
- **Built-in audit trail** — every provisioning attempt (success or failure) is logged to Airtable with a timestamp.
- **Immediate visibility** — IT/Ops and stakeholders get a Slack + email notification the moment provisioning completes, and a separate Slack alert the moment something fails.
- **Scales without added headcount** — adding a new department or employee doesn't require more manual admin work.

## Who Uses It

- **HR** — adds/updates the new hire's row in the source Google Sheet; that's their only manual step.
- **IT/Ops** — receives Slack notifications (both success confirmations and error alerts) and uses the Airtable log as the system of record for provisioning status.
- **Hiring managers** — indirectly benefit from faster account readiness for their new hires, and receive the summary email notification.
- **New hires** — the ultimate beneficiary; their accounts are ready sooner and more reliably.

## Architecture Overview

```
                                   ┌─────────────────────────────┐
                                   │      Google Sheet           │
                                   │   "New Employee" tab        │
                                   └──────────────┬───────────────┘
                                                  │ new/updated row
                                                  ▼
                                   ┌─────────────────────────────┐
                                   │   fetchDataFromSheets        │
                                   │   (Google Sheets Trigger)    │
                                   └──────────────┬───────────────┘
                                                  ▼
                                   ┌─────────────────────────────┐
                                   │        mapData (Set)         │
                                   │  normalize fields, generate  │
                                   │      temp password            │
                                   └──────────────┬───────────────┘
                                                  ▼
                                   ┌─────────────────────────────┐
                                   │   mapDepartment (Switch)     │
                                   │   routes on Department       │
                                   └───┬───────┬───────┬─────┬───┘
                          Engineering  │   Sales   Marketing │ Human Resources
                                       ▼       ▼         ▼    ▼
                        ┌──────────┐┌──────────┐┌──────────┐┌──────────┐
                        │engineering││  sales   ││marketing ││   HR     │
                        │MockProvis.││MockProvis││MockProvis││MockProvis│
                        │(HTTP → sub-workflow webhook)                  │
                        └────┬─────┘└────┬─────┘└────┬─────┘└────┬─────┘
                             │  success/error x4 (error → errorMessage)
                             ▼
                     ┌─────────────────────┐
                     │  mapDataOutput        │
                     │  (Merge, Append, 4in) │
                     └───────────┬────────────┘
                                 ▼
                     ┌─────────────────────┐
                     │  editDataOutput (Set)│
                     │  reshape for logging  │
                     └───────────┬────────────┘
                                 ▼
                     ┌─────────────────────┐
                     │  recordData (Airtable)│
                     │  Upsert on Employee ID│──error──► errorMessage (Slack)
                     └───────────┬────────────┘
                                 ▼
                    ┌────────────┴────────────┐
                    ▼                          ▼
        ┌───────────────────┐      ┌───────────────────┐
        │ slackNotifications │      │  Send a message    │
        │  (Slack success)   │      │  (Gmail summary)   │
        └────────────────────┘      └────────────────────┘
```

Each of the four `*MockProvisioning` HTTP Request nodes calls out to its **own separate, published n8n sub-workflow** (`Engineering-Mock-Provisioning`, `Sales-Mock-Provisioning`, `Marketing-Mock-Provisioning`, `HR-Mock-Provisioning`), each with a Webhook trigger and a Respond to Webhook node that echoes back a simulated success response.

## Data Schema (Source Sheet)

The trigger watches a Google Sheet tab named **"New Employee"** with the following columns:

| Column | Type | Example | Notes |
|---|---|---|---|
| Employee ID | Text | `EMP011` | Unique identifier, used as the Airtable upsert key |
| Name | Text | `Juan Dela Cruz` | Full name; split into first/last downstream |
| Email | Text | `juan.cruz.personal@gmail.com` | Personal contact email |
| Phone | Text | `0917-556-7789` | Not currently used in the pipeline logic |
| Address | Text | `14 Timog Ave, QC` | Not currently used in the pipeline logic |
| Position | Text | `QA Engineer` | Included in provisioning payload and notifications |
| Department | Text | `Engineering` | Drives the Switch node routing — must exactly match one of: `Engineering`, `Sales`, `Marketing`, `Human Resources` |
| Date Hired | Date | `2026-08-11` | Not currently used in the pipeline logic |
| Company Email | Text | `juan.cruz@company.com` | The generated work email used for provisioning |
| Status | Text | `Pending` | Reserved for future use (e.g. skip already-provisioned rows) |
| Tools Required | Text | `Slack, GitHub, Jira, AWS` | Informational; not currently used for routing (routing is on Department directly) |

> **Note:** the sheet is *not* written back to by this workflow — all output goes to a separate Airtable base — to avoid the trigger re-firing on its own writes (a self-triggering loop).

## Prerequisites & Tech Stack

- An **n8n instance** (self-hosted or cloud) with the ability to publish/activate multiple workflows
- **Google Sheets** — source of new hire data, connected via OAuth2 credential
- **Airtable** — provisioning log destination, connected via Personal Access Token credential
- **Slack** — a Slack App/bot with a channel it has been invited to, connected via Slack API credential (needs `chat:write` scope at minimum)
- **Gmail** — connected via OAuth2 credential, used to send the new-hire announcement email
- No paid external HR/IT tools (Google Workspace Admin, Jira, GitHub, etc.) are required — those integrations are currently **mocked** via internal sub-workflows

## Setup & Installation

1. **Import all 5 workflow JSON files** into your n8n instance:
   - `New_User_Account_Provisioning_Pipeline.json` (main workflow)
   - `Engineering-Mock-Provisioning.json`
   - `Sales-Mock-Provisioning.json`
   - `Marketing-Mock-Provisioning.json`
   - `HR-Mock-Provisioning.json`
2. **Publish/activate the 4 mock sub-workflows first** so their webhook URLs are live.
3. In the main workflow, confirm each `*MockProvisioning` HTTP Request node's URL matches the corresponding sub-workflow's **Production** webhook URL (not the Test URL — Test URLs only fire once per manual "Listen" click).
4. **Reconnect credentials** for each of the following nodes (credentials are not portable between n8n instances/accounts and must be re-selected after import):
   - `fetchDataFromSheets` — Google Sheets Trigger OAuth2
   - `recordData` — Airtable Personal Access Token
   - `slackNotifications` / `errorMessage` — Slack API
   - `Send a message` — Gmail OAuth2
5. Point `fetchDataFromSheets` at your own copy of the source sheet (Document ID + Sheet Name), and `recordData` at your own Airtable base/table.
6. Update the Slack `channelId` in `slackNotifications` and `errorMessage` to your own workspace's channel, and make sure the bot is invited to it (`/invite @YourBotName`).
7. Update the Gmail node's `sendTo` address to whoever should receive the new-hire announcement.
8. **Activate the main workflow.**
9. Add a test row to the source sheet and confirm the full pipeline runs (see [Testing the Pipeline](#testing-the-pipeline)).

## Main Workflow — Node by Node

| Node | Type | Function |
|---|---|---|
| **fetchDataFromSheets** | Google Sheets Trigger | Polls the source "New Employee" sheet every minute and fires whenever a row is added or updated. This is the entry point of the whole pipeline. |
| **mapData** | Set (Edit Fields) | Normalizes the raw sheet row into clean, consistently-named fields: Employee ID, Name, Company Email, Position, Status, Tools Required, Department, and a randomly generated temporary password (`Math.random().toString(36).slice(-8) + "!A1"`). |
| **mapDepartment** | Switch | Routes each employee into one of four branches based on an **exact match** on their `Department` field: Engineering, Sales, Marketing, or Human Resources. (Originally used a "contains" match, which was tightened to "equals" for reliability.) |
| **engineeringMockProvisioning** | HTTP Request (POST) | Sends the employee's payload (email, first/last name, department, position) to the Engineering mock provisioning endpoint. Has a separate error output (`onError: continueErrorOutput`) so a failure here doesn't halt the other branches. |
| **salesMockProvisioning** | HTTP Request (POST) | Same as above, sent to the Sales mock endpoint. |
| **marketingMockProvisioning** | HTTP Request (POST) | Same as above, sent to the Marketing mock endpoint. |
| **HRMockProvisioning** | HTTP Request (POST) | Same as above, sent to the Human Resources mock endpoint. |
| **mapDataOutput** | Merge (Append, 4 inputs) | Recombines all four department branches back into a single stream, so each employee produces exactly one downstream item regardless of which branch they took. |
| **editDataOutput** | Set (Edit Fields) | Reshapes the merged response data into a clean, consistent record for logging: Employee ID (pulled back from the original `mapData` item, since it isn't part of the mock response), Name, Email, Department, Position, Status. |
| **recordData** | Airtable (Upsert) | Writes the provisioning result to the **"Account Provisioning"** base's **"User Data"** table, matching/upserting on `Employee ID` so re-runs update the existing record instead of creating duplicates. Has a separate error output feeding into `errorMessage`. |
| **slackNotifications** | Slack (Post Message) | Posts a success message to the IT/Ops Slack channel confirming the new hire's accounts were provisioned, using the just-written Airtable record's fields. |
| **Send a message** | Gmail (Send) | Sends a plain-text email announcing the new hire, their position, and department. |
| **errorMessage** | Slack (Post Message) | Centralized error handler — receives the error output from all four provisioning nodes **and** the Airtable node, and posts a detailed failure alert (employee, department, failed step, error message, timestamp) to the same Slack channel. |

## Sub-Workflows — Mock Provisioning Endpoints

There are four nearly identical sub-workflows, one per department (`Engineering-Mock-Provisioning`, `Sales-Mock-Provisioning`, `Marketing-Mock-Provisioning`, `HR-Mock-Provisioning`). Each one stands in for a real external system's account-creation API and is called via webhook from the main workflow's HTTP Request nodes.

| Node | Type | Function |
|---|---|---|
| **mockTrigger** (named `mocktrigger` in the Marketing sub-workflow) | Webhook (POST) | Listens for the POST sent by the corresponding HTTP Request node in the main workflow, carrying the new hire's payload (email, firstName, lastName, department, position). Response mode is set to "Using Respond to Webhook Node" rather than immediate response. |
| **respondToMainWorkflow** (named `Respond to Webhook` in the HR and Marketing sub-workflows) | Respond to Webhook | Builds and returns a JSON confirmation: `status: "success"`, the department name, a `message`, and `receivedData` — an echo of the request body via `JSON.stringify($json.body)`, simulating what a real provisioning API would send back on success. |

Because each department's mock sub-workflow is a standalone, published workflow with its own webhook URL, any one of them can be swapped out for a real integration (Google Workspace Admin, Slack invite, GitHub/Jira account creation, etc.) later without needing to change anything else in the main pipeline — the main workflow only cares that it gets back a `status`/`department`/`message`/`receivedData` shaped response.

## Error Handling Design

Rather than a single global "Error Workflow" (n8n's built-in top-level error handler), this pipeline uses **per-node error outputs**, wired to a shared Slack alert node:

- Each of the 4 `*MockProvisioning` HTTP Request nodes and the `recordData` Airtable node has `onError: continueErrorOutput` set, giving them two outputs: **success** and **error**.
- All 5 error outputs converge on the same **`errorMessage`** Slack node.
- This means:
  - One department's provisioning call failing does **not** stop the other three branches from completing.
  - A failure at the Airtable logging step still gets flagged, even though provisioning itself succeeded.
  - IT gets a single, consistent alert format regardless of which step failed.

The error message template used:

```
🚨 Provisioning Error
Employee: {{ $('mapData').item.json.Name }}
Department: {{ $('mapData').item.json.Department }}
Failed Step: {{ $json.node?.name || "Unknown" }}
Error: {{ $json.error?.message || $json.message }}
Time: {{ $now }}
```

Employee/Department are pulled back from the original `mapData` node (rather than the failed node's own output) using `$('mapData').item.json`, since the shape of an error item can vary depending on which node failed.

## Airtable Log Schema

Base: **Account Provisioning** → Table: **User Data**

| Field | Type | Source | Purpose |
|---|---|---|---|
| Employee ID | String | `mapData` (passed through) | Upsert matching key — prevents duplicate records on re-runs |
| Name | String | Mock response `receivedData` | Full name (first + last) |
| Email | String | Mock response `receivedData` | Company email |
| Department | String | Mock response `receivedData` | Department the employee was routed to |
| Position | String | Mock response `receivedData` | Job title |
| Status | String | Mock response `status` | `success` (or reflects the mock endpoint's returned status) |

## Notifications

Two notifications fire in parallel after a successful Airtable write:

1. **Slack** (`slackNotifications`) → posted to the `it-ops-n8n-test` channel:
   ```
   ✅ New hire provisioned: {{ Name }}
   Department: {{ Department }}
   Status: {{ Status }}
   ```
2. **Gmail** (`Send a message`) → sent to a designated recipient with subject "New Hire Provisioned!" and a short welcome message including the new hire's name, position, and department.

A separate error notification (see [Error Handling Design](#error-handling-design)) fires independently to the same Slack channel whenever something fails instead of succeeding.

## Security Notes

- The temporary password generated in `mapData` (`temPassword`) is created in-memory for use in a real provisioning API call — it is **not currently written to Airtable or included in any Slack/Gmail message**, which is intentional. If real provisioning integrations are added later, make sure this stays true: never let a plaintext temporary password land in a log sheet or an unencrypted notification channel.
- Credentials (Google Sheets, Airtable, Slack, Gmail) are stored in n8n's credential store, not hardcoded in node parameters.
- Mock endpoints currently accept unauthenticated POST requests — acceptable for an internal testing/demo setup, but real production endpoints should require authentication (API key/OAuth) before going live.

## Testing the Pipeline

**Happy path test:**
1. Add a new row to the source sheet with a valid `Department` (exact match: `Engineering`, `Sales`, `Marketing`, or `Human Resources`).
2. Wait for the next poll (up to 1 minute) or trigger manually.
3. Confirm: the correct department branch fires → Airtable gets a new/updated row → Slack success message appears → email arrives.

**Failure path test:**
1. Temporarily deactivate one of the mock sub-workflows (e.g. `Sales-Mock-Provisioning`).
2. Add/update a row for that department.
3. Confirm: the `errorMessage` Slack alert fires with the correct employee/department, and — separately — confirm other departments' rows still process normally and aren't affected.
4. Reactivate the sub-workflow afterward.

**Airtable error test:**
1. Temporarily rename or remove a mapped field in the Airtable table.
2. Run a test row through.
3. Confirm the Airtable node's error output also correctly routes into `errorMessage`.

## Troubleshooting

Issues encountered and resolved during development, kept here for reference:

| Symptom | Cause | Fix |
|---|---|---|
| `The resource you are requesting could not be found` / "webhook not registered" | Using a sub-workflow's **Test URL** without clicking "Listen for Test Event" first, or the Test listener had already expired after catching one request | Use the **Production URL** on an **activated** sub-workflow instead, or re-click "Listen for Test Event" before every test run |
| Same "not registered" error even after re-arming Listen | Webhook node's **Method** (GET) didn't match the HTTP Request node's Method (POST) | Set the Webhook node's Method to match exactly (POST) |
| `receivedData` showed the literal text `{{ $json }}` | The field was in **Fixed** mode instead of **Expression** mode in the Respond to Webhook node | Toggle the field to Expression mode (fx icon) |
| `receivedData` showed `[object Object]` | `{{ $json }}` was inserted directly into a JSON string field — JavaScript's default `toString()` on an object returns `"[object Object]"` | Use `{{ JSON.stringify($json.body) }}` instead |
| Response came back completely empty | The whole Response Body field got wrapped as a single top-level expression, or the expression wasn't valid JSON syntax | Keep the JSON body as literal text with `{{ JSON.stringify($json.body) }}` inlined as a raw (unquoted) value, not the whole block as one expression |
| Slack error: `not_in_channel` | The Slack bot has `chat:write` permission but was never added to the target channel | Run `/invite @YourBotName` in the channel, or add the `chat:write.public` scope and reinstall the app |
| `Unused Respond to Webhook node found in the workflow` | Either the Webhook node's "Respond" setting was still on "Immediately" instead of "Using Respond to Webhook Node", or the Respond to Webhook node existed but wasn't actually connected | Set Webhook → Respond to "Using Respond to Webhook Node" and ensure a visible connection line exists between the two nodes |
| Google Workspace Admin node needs a paid plan | Google Workspace requires a paid subscription to use the Admin SDK; no free tier supports user provisioning | Use a mock sub-workflow (current approach) for testing, or a Google Workspace free trial / Zoho Mail free plan (needs a custom domain) if a real integration is needed |

## Known Limitations

- **Provisioning is mocked**, not real — the 4 department endpoints simulate success but don't actually create accounts in Google Workspace, Slack, GitHub, Jira, or any other real system yet.
- **Only one tool call per department branch** — the original plan called for multiple tool-specific calls per department (e.g. Engineering needing separate Slack, GitHub, Jira, and Google Workspace calls); the current build simplifies this to a single representative mock call per department.
- **No fallback/default route** in the Switch node — a row with a `Department` value that doesn't exactly match one of the four expected values will be silently dropped with no output and no error alert.
- **`Status` and `Tools Required` columns are not yet used** by the pipeline logic (e.g. to skip already-provisioned rows or dynamically decide which tools to call).
- **Real Slack account invites are not implemented** — Slack's `admin.users:write` scope (true workspace invites) requires Slack Enterprise Grid; the current Slack node only posts messages, which works on any plan.

## Roadmap / Future Improvements

- Replace mock sub-workflows with real integrations (Google Workspace Admin node, native Slack/GitHub/Jira nodes) one department/tool at a time.
- Expand each department branch to fan out into multiple tool-specific calls (per the original `Tools Required` column) instead of one combined mock call.
- Add a Switch fallback/default output to catch unexpected or mistyped `Department` values and alert instead of silently dropping the row.
- Use the `Status` column to skip rows that have already been successfully provisioned, preventing redundant re-processing on every poll.
- Add a `Wait` step to delay provisioning until closer to the employee's actual start date, rather than immediately on row creation.
- Add a scheduled follow-up (e.g. 90 days later) for probation review or access-audit tasks.

## FAQ

**What is the benefit of this workflow in a company?**
See [Why This Exists](#why-this-exists) above — primarily time savings, consistency, an audit trail, and faster new-hire readiness.

**What problem does it solve?**
The manual, error-prone, and inconsistent process of creating new-hire accounts across multiple department-specific tools by hand.

**Who will use it?**
HR (data entry), IT/Ops (monitoring and the audit log), hiring managers (notifications), and indirectly, new hires themselves.

**Give a short summary description of this workflow.**
See [Summary](#summary) at the top of this document.

**Explain node by node their function, as well as the sub-workflows.**
See [Main Workflow — Node by Node](#main-workflow--node-by-node) and [Sub-Workflows — Mock Provisioning Endpoints](#sub-workflows--mock-provisioning-endpoints) above.
