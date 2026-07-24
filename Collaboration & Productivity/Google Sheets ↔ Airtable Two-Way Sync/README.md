# Google Sheets ↔ Airtable Two-Way Sync (n8n)

A practice n8n workflow that keeps a Google Sheet and an Airtable base in sync in both directions.

## Overview

This workflow is split into two independent paths that run in parallel:

| Path | Trigger | Direction |
|---|---|---|
| **Airtable to Google Sheet** | Airtable Trigger (polling) | Airtable → Sheets |
| **Google Sheet to Airtable** | Schedule Trigger (every 1 min) | Sheets → Airtable |

Both paths share the same table structure so records can be matched across platforms using a common `Record ID`.

## Data model

Both the Airtable base and the Google Sheet use identical columns:

| Column | Type (Airtable) | Notes |
|---|---|---|
| Name | Single line text | |
| Email | Email | |
| Status | Single select (Active / Inactive / Pending) | |
| Amount | Number | |
| Last Update | Date (with time) | see [Known limitations](#known-limitations) |
| Record ID | Formula: `RECORD_ID()` | read-only, used as the sync key on both sides |

The Google Sheet has the same header row, with `Record ID` populated from Airtable so both platforms can match rows to each other.

## Architecture

```
🔴 Airtable to Google Sheet
[Airtable Trigger] → [Edit Fields: flatten] → [Google Sheets: Append or Update Row]

🟢 Google Sheet to Airtable
[Schedule Trigger: every 1 min] → [Google Sheets: Get rows] → [Edit Fields: flatten] → [Airtable: Update record]
```

### Path 1 — Airtable → Google Sheet

1. **Airtable Trigger** — polls the base for changes to the `Last Update` field
   - Credential: Airtable Personal Access Token
   - Base / Table: set via "By ID" (paste the `app...` ID from the Airtable URL)
   - Trigger Field: `Last Update`
2. **Edit Fields (Set)** — flattens the nested `fields.*` object from Airtable's response into flat keys: `Name`, `Email`, `Status`, `Amount`, `Last Update`, `Airtable Record ID` (mapped from `id`)
3. **Google Sheets — Append or Update Row**
   - Matching Column: `Record ID`
   - Maps all flattened fields into the sheet

### Path 2 — Google Sheet → Airtable

1. **Schedule Trigger** — every 1 minute
2. **Google Sheets — Get Row(s)** — pulls all rows from the sheet (Return All on)
3. **Edit Fields (Set)** — normalizes/flattens the row data if needed
4. **Airtable — Update record**
   - Columns to match on: `Record ID`
   - Fields to Send: manually mapped — `Name`, `Email`, `Status`, `Amount`, `Last Update` (Record ID is **not** included here, since it's a computed/read-only field in Airtable)
   - Typecast: enabled (handles type mismatches, e.g. Sheets sending dates as plain text)

## Credentials setup

### Airtable
1. Airtable → profile icon → **Developer Hub** → **Personal access tokens** → Create new token
2. Scopes: `data.records:read`, `data.records:write`, `schema.bases:read`
3. Access: scope to the specific base (or all bases, if you plan to reuse the token across future workflows)
4. In n8n: Credentials → New → **Airtable Personal Access Token API** (the old "Airtable API Key" type is deprecated)

### Google Sheets
1. In n8n: Credentials → New → **Google Sheets OAuth2 API** → sign in and grant access

## Known limitations / next steps

- **`Last Update` is a manually-typed field.** The Airtable Trigger only fires when this field changes — if someone edits a record but forgets to update this field, Path 1 won't catch it. Fix: convert it to Airtable's built-in **Last Modified Time** field type so it updates itself automatically.
- **Path 2 pushes every row on every poll**, not just changed ones. At small scale (a few test rows) this is harmless, but it doesn't scale well. Fix: add an IF/Filter node after `Get Rows` that only passes rows where `Last Update` falls within the last poll interval.
- **No loop-prevention flag yet.** Since both paths currently write based on a `Last Update` timestamp rather than an explicit "synced by" flag, there's a small theoretical risk of the two paths re-triggering each other. For this practice build with manual testing it hasn't caused issues, but a production version would add a "Synced From Sheets" checkbox in Airtable to break the loop explicitly.

## Notes

Built as a hands-on training exercise, based on a two-way sync pattern from a "top 100 n8n workflows" list. Not connected to any other workflow.
