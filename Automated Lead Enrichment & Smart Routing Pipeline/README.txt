🚀 Automated Lead Enrichment & Smart Routing Pipeline

An advanced, batch-optimized **n8n automation workflow** designed to ingest inbound lead submissions, check for existing database records, score profile matches, enrich missing data points using external APIs, and route leads dynamically based on value tiers.

---

🛠️ System Architecture Overview

The pipeline handles data linearly before branching into two specialized processing tracks to prevent data drops, optimize API consumption, and maximize sales efficiency.

```text
               ┌───────────┐    ┌────────────┐    ┌─────────────┐
               │ Test Data │───►│ Get Domain │───►│ CRM Search  │
               └───────────┘    └────────────┘    └─────────────┘
                                                         │
                                                         ▼
┌──────────────────────┐    ┌──────────────────┐    ┌───────────┐
│ Duplicate Detector   │◄───│ Loop Fixer (JS)  │◄───│  (Zips)   │
└──────────────────────┘    └──────────────────┘    └───────────┘
     │               │
     ▼ (True)        ▼ (False)
 [ CRM Duplicates ]   [ New Leads Lane ]

```

---

📝 Node-by-Node Documentation

 1. Ingestion & Data Preparation

* Trigger Node: Kicks off the execution flow. In production, this is swapped for an **Inbound Webhook** or active form listener.
* Test Data Node: Ingests raw mock data objects (e.g., 10 entries) containing essential identifiers like `First Name`, `Last Name`, and `Email`.
* Get Domain Node: Strips email strings down to their core corporate domains (e.g., `enterprise.com`) to enable high-level organizational mapping.

 2. Identity Resolution & State Recovery

* CRM Search (HTTP Request): Performs a dynamic lookup against the historical database using the lead's email. New users safely return empty objects `{}`.
* Code in JavaScript (Loop Fixer): > **Critical Failure Prevention:** n8n natively drops remaining batch structures when dynamic HTTP nodes encounter an empty result. This 3-line JavaScript block zips the original submission data with the CRM output to preserve array structures across all elements.
* Duplicate Detector (Router): Evaluates if a CRM match exists based on `crm_match.id`.
* True Branch: Passes existing accounts to the CRM Duplicates route.
* False Branch: Passes brand-new leads to the Enrichment lane.



---

 🔀 Processing Lanes

  🟢 Lane A: CRM Duplicates (Top Track)

Designed to surface returning clients without burning extra API credits on data enrichment.

* High Value Duplicate?: Scans existing CRM profile metadata using a Regular Expression signature match (`Enterprise|Mid-Market|Tier 1`).
* Hot Action Paths: High-value duplicates trigger an immediate Slack/Teams **Message to Team** notification, while low-value duplicates automatically route to a baseline nurture sheet.

  🔵 Lane B: New Leads Lane (Bottom Track)

Designed to score, evaluate, and provision newly discovered market opportunities.

  🧮 Lead Scorer Algorithm Logic

The JavaScript scoring block processes every incoming item dynamically across a $100$-point scale:

$$\text{Final Score} = 50 \,(\text{Base}) + \Delta_{\text{Domain}} + \Delta_{\text{Email}} + \Delta_{\text{Location}}$$

* Rule 1 (Firmographic Match): Corporate extensions (`.org`, `.net`) or specific enterprise keywords add **+35 points**.
* Rule 2 (Spam/Generic Filter): Public email addresses (`gmail.com`, `yahoo.com`) subtract **-40 points**.
* Rule 3 (Geographic Bonus): Verified city data locations add **+10 points**.

```text
         Score >= 80   ──►  Tier 1 (Enterprise)
   40 <= Score <= 79   ──►  Tier 2 (Mid-Market)
         Score < 40    ──►  Tier 3 (Low Priority / Junk)

```

 🔌 Enrichment & Tiering

* Data Enrichment (Clearbit/ZoomInfo Mock): Queries third-party APIs to backfill firmographic data for newly qualified leads.
* Tier Classifier (Switch Node): Reads `{{ $json.scoring_metrics.final_score }}` and splits execution pipelines natively.
* CRM Provisioning: Provision high/mid tiers directly into active pipeline databases (Salesforce / Hubspot / Pipedrive) while dropping Tier 3 spam leads into an archive table.

---

🚀 Deployment & Local Testing

1. Import Workflow: Copy your workspace JSON and paste it directly onto an empty n8n canvas.
2. Environment Targets: Update the placeholder URLs in the **CRM Search** and **Enrichment** blocks to point to your live system endpoints.
3. Execute: Click *Execute workflow* on the main navigation panel to monitor the data streams across the routing tables in real-time.