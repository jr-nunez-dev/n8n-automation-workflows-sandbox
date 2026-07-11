# Dynamic Lead Enrichment & Smart Geo-Router

## Overview
This automated n8n production workflow manages inbound B2B and B2C lead qualification. It screens out generic public signups, enriches high-intent corporate profiles via parallel REST API lookups, applies a programmatic valuation score, and dynamically routes high-value opportunities to specific geographic sales queues (US vs. EU) while seamlessly preserving system uptime during third-party API outrages.

## Workflow Mechanics

1. INBOUND FILTERING
   - Captures incoming lead payloads.
   - Evaluates email syntax. Public domains (e.g., Gmail) instantly route to standard marketing nurture paths and post operational Slack logs. 
   - Verified business emails move immediately into premium processing.

2. PARALLEL ENRICHMENT
   - Simultaneously queries independent mock data endpoints (`/users` and `/todos`) using parsed numeric extractions derived from the unique `lead_id`.
   - Utilizes balanced modulo syntax (`% 10 + 1`) to ensure 1:1 records sync cleanly across concurrent HTTP request operations without generating 404 string errors.

3. EMBEDDED SYSTEM FAILSAFE (ENTERPRISE RESILIENCY)
   - Connects HTTP node error outputs directly to an error recovery sequence.
   - If a vendor API fails, crashes, or returns an un-parsable payload, the workflow silently absorbs the error, defaults the lead profile to a safe baseline score of 10, caches the user data safely to the Nurture Queue, and shoots an emergency tracking notification to internal engineering channels via Slack.

4. MERGE & COMPRESSION SCORING
   - The Enrichment Combination node synchronizes the data tracks.
   - Applies an evaluation formula based on metadata existence:
     `{{ $json.company && $json.company.catchPhrase ? ($json.company.catchPhrase.length * 150 > 5000 ? 100 : 50) : 10 }}`
   - Segregates verified data payloads into precise Switch channels (Score 100 / Score 50 / Score 10).

5. GEO-TARGETED CONVERSION ROUTING
   - Evaluates geographic target zones (US vs. EU).
   - Bundles relevant payloads via dedicated data compression nodes (`UScompressData` / `EUcompressData`).
   - Dispatches hot leads instantly to active region-specific Slack alert monitors so account reps can respond immediately.