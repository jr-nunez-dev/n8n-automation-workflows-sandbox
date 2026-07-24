# 🏆 Hackathon Finisher Certificate Bot

An n8n workflow that automatically generates and emails personalized "You Survived 48 Hours" completion certificates to hackathon participants, with a random MVP Award bonus and a live Slack shoutout for the winner.

---

## Summary

This workflow reads a list of hackathon participants (name, email, rank) from a Google Sheet, randomly selects one lucky participant as the "MVP" for the run, generates a personalized PDF certificate for every participant, emails each person their certificate via Gmail, and posts a Slack announcement celebrating the MVP winner. It runs end-to-end with no manual data entry or manual certificate design required.

---

## What problem does it solve?

Manually creating and distributing completion certificates after an event is repetitive, slow, and error-prone — someone has to open a design template, swap in each person's name and rank, export it, and email it one by one. For an event with even a modest number of participants, this becomes a real time sink right when the organizing team is already busy wrapping up the event.

This workflow removes that entire manual step. Once the participant list is in a spreadsheet, the certificates generate and send themselves — correctly personalized, every time, with zero copy-paste errors.

---

## What's the benefit to a company?

- **Time savings** — turns what could be an hour-plus of manual certificate creation and emailing into a workflow that runs unattended.
- **Consistency** — every certificate follows the exact same template and formatting; no risk of a typo'd name or mismatched rank slipping through.
- **Team morale / engagement** — the random MVP Award adds a small moment of surprise and recognition, reinforced publicly in Slack, which helps close out events on a positive, memorable note at negligible cost.
- **Reusable pattern** — the same structure (Sheet → personalize → generate document → email) is a template that can be adapted for other automated-document scenarios: training completion certificates, onboarding welcome packets, event badges, etc.
- **Scales for free** — whether 5 participants or 50, the workflow handles it the same way, with no additional manual effort.

---

## Who will use it?

- **Hackathon / event organizers** — the primary users; they maintain the participant sheet and trigger the run after an event concludes.
- **People/Culture or Employee Engagement teams** — could repurpose this for internal recognition programs, training completions, or team-building events.
- **Hackathon participants** — the end recipients, who receive their certificate by email and may be publicly recognized in Slack if selected as MVP.

---

## Architecture Overview

```
trigger (Schedule)
   → getData (Google Sheets)
      → pickMVP (Code — randomly flags one participant as MVP)
         → loopData (Split In Batches — loops one participant at a time)
              ├──► isMVP? (IF)
              │      ├─ true  → MVPmessage (Slack announcement)
              │      └─ false → notMVP (No-Op)
              │
              └──► generateCert (Code — builds certificate HTML)
                     → convertCert (HTML to PDF)
                        → getPDFCert (HTTP Request — downloads PDF as binary)
                           → Send a message (Gmail — emails cert with attachment)
                              ↳ loops back into loopData for the next participant
```

Each loop iteration does two things in parallel for the current participant: it checks whether they're this run's MVP (posting to Slack if so) and it generates + emails their certificate. Once the certificate email is sent, the loop advances to the next row in the sheet, repeating until every participant has been processed.

---

## Prerequisites

**Google Sheet** — a sheet (e.g. "Hackathon Participants") with a tab containing at minimum these columns:

| Name | Email | Rank |
|------|-------|------|
| Juan Dela Cruz | juan@example.com | 1 |
| ... | ... | ... |

**Credentials required:**
- Google Sheets OAuth2 — to read the participant list
- HTML-to-PDF API account (htmlcsstopdf) — to render the certificate HTML into a downloadable PDF
- Gmail OAuth2 — to send certificate emails
- Slack API — to post the MVP announcement

---

## Node-by-Node Breakdown

### `trigger` — Schedule Trigger
Kicks off the workflow on a set interval. This is the entry point — no manual click or mock data required. In production, the interval should be set to match how often you actually want this to run (e.g. once, right after an event ends), rather than the short testing interval used during development.

### `getData` — Google Sheets
Reads all participant rows from the configured spreadsheet/tab. Each row (Name, Email, Rank) becomes one item flowing into the rest of the workflow — this is the single source of truth for who receives a certificate.

### `pickMVP` — Code
Runs once across the entire participant list (not per item) and uses `Math.random()` to select exactly one random index as this run's MVP. It tags every item with:
- `isMVP` (`true` for the winner, `false` for everyone else)
- `badge` (a label string, populated only for the winner)

Running this once across all items — rather than per item — is what guarantees exactly one winner per run instead of a different random result on every loop iteration.

### `loopData` — Split In Batches
Takes the full participant list and processes it one item at a time (batch size 1). This is what turns a 5-item list into 5 sequential iterations, ensuring each participant's certificate/email/Slack check is handled individually rather than all at once. It also serves as the loop's re-entry point — the workflow routes back here after each participant is fully processed, advancing to the next one.

### `isMVP?` — IF
Checks the current item's `isMVP` flag.
- **True branch** → routes to `MVPmessage` (Slack).
- **False branch** → routes to `notMVP` (a no-op — nothing happens for non-winners on this branch).

### `MVPmessage` — Slack
Posts an announcement to a configured Slack channel celebrating the MVP winner by name and rank, giving the team a real-time, visible moment of recognition.

### `notMVP` — No-Op
A deliberate do-nothing node for the non-MVP path of the IF branch — keeps the workflow structure clean without needing extra logic for the common case.

### `generateCert` — Code
Runs once per item (per participant) and builds a fully personalized HTML certificate string, injecting the participant's name, rank, and — if applicable — the MVP Award callout. This HTML is the actual certificate design/template, stored inline as a template literal.

### `convertCert` — HTML to PDF
Sends the generated certificate HTML to a third-party HTML-to-PDF rendering API and receives back a URL pointing to the generated PDF file.

### `getPDFCert` — HTTP Request
Downloads the PDF from the URL returned by `convertCert`, converting it into binary file data (stored in the `data` field) so it can be attached to an email — a URL alone isn't something Gmail can attach directly.

### `Send a message` — Gmail
Sends the personalized email to the participant, including:
- Subject and message body referencing their name and rank (pulled back from `generateCert`'s output, since the binary-handling nodes in between don't carry the original participant fields)
- A conditional MVP congratulations line, shown only if `isMVP` is true
- The certificate PDF attached, using the binary field `data` from `getPDFCert`

After sending, this node loops back into `loopData`, triggering the next participant's iteration.

---

## The "Bonus Thrill" — Random MVP Logic

The MVP selection is intentionally computed **once per workflow run, across the whole batch**, in the `pickMVP` node — not inside the per-participant loop. This is what makes it a true "1 random winner per run" mechanic rather than a coin-flip that could theoretically tag multiple (or zero) participants. That single `isMVP` flag then drives two independent downstream effects for the winner: an extra congratulatory section on their certificate/email, and a public Slack shoutout.

---

## Known Considerations

- The PDF files generated by the HTML-to-PDF service are hosted temporarily (auto-deleted after a set retention window) — fine for immediate emailing, but not a permanent archive of certificates.
- The Slack (`MVPmessage`/`notMVP`) branch runs in parallel to the certificate/email branch but does not itself loop back into `loopData` — only the Gmail send does. Since n8n's Split In Batches only needs one path to signal completion to advance, this works, but it does mean the Slack post isn't guaranteed to finish before the loop advances to the next participant. Worth keeping in mind if exact ordering ever matters.
