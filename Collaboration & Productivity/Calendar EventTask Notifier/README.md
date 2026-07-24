# 📅 Calendar Pulse — Event & Task Slack Notifier

An n8n workflow that polls Google Calendar on a schedule and pushes real-time Slack notifications, automatically distinguishing between calendar **events** and **task-type** entries.

## What it does

Every 5 minutes, this workflow checks your Google Calendar for upcoming items and posts a formatted message to Slack — routing standard events and task-style entries to differently worded notifications, so your team never misses a meeting or a deadline.

## How it works

```
every5Mins (Schedule Trigger)
        │
        ▼
   getEvents (Google Calendar — Get Many)
        │
        ▼
   eventType (If node)
   ├── true  → eventMessage  (Slack)
   └── false → taskMessage   (Slack)
```

| Node | Type | Purpose |
|---|---|---|
| `every5Mins` | Schedule Trigger | Runs the workflow on a 5-minute interval |
| `getEvents` | Google Calendar (Get Many) | Pulls up to 10 upcoming calendar entries within the lookahead window |
| `eventType` | If | Checks `eventType == "default"` to split standard events from everything else |
| `eventMessage` | Slack | Posts a "new event" notification with title, description, start/end time |
| `taskMessage` | Slack | Posts a "task due" notification with title, deadline, and organizer sign-off |

## Setup

1. Import `Calendar Event_Task Notifier clean.json` into n8n.
2. Connect your **Google Calendar OAuth2** credential and select the target calendar in the `getEvents` node.
3. Connect your **Slack API** credential and set the destination channel in both `eventMessage` and `taskMessage`.
4. Set the workflow timezone (this one is configured for `Asia/Manila`) under **Settings**.
5. Activate the workflow.

## Sample output

**Event notification:**
```
This is an Event Notification!

Title: Sprint Planning
Message: Review backlog for next sprint
When: July 23, 2026 : 10:00 AM
Until: July 23, 2026 : 11:00 AM
```

**Task notification:**
```
This is a Task Notification:

Title: Submit Q3 Report

Team,

Kindly accomplish your tasks on or before July 25, 2026 : 05:00 PM.

Regards,
johnrex
```

## Known limitations / next steps

- No dedupe logic yet — if the lookahead window is wider than the polling interval, the same event can be notified more than once. Recommended fix: set `timeMin`/`timeMax` to match the 5-minute polling interval, or add an "already notified" tracking step.
- `eventType == "default"` is used as a proxy for "task vs. event" — Google Calendar's actual event types (`focusTime`, `outOfOffice`, etc.) aren't explicitly handled, so non-default types other than tasks will currently route to the task branch.
- **Planned:** an AI Agent version that classifies and summarizes calendar entries dynamically instead of relying on a single field check.
