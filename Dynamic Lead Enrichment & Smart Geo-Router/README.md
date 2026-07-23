\# AI Lead Enrichment, Scoring \& Regional Sales Routing Workflow



An intelligent n8n workflow that automatically evaluates inbound leads, enriches company information, calculates a lead score, classifies prospects into sales tiers, routes qualified leads to the appropriate regional sales team, and sends notifications while directing low-value leads into a nurture pipeline.



\---



\## Quick Facts



| Item | Value |

|------|------|

| \*\*Workflow Type\*\* | AI Sales Operations Automation |

| \*\*Primary Purpose\*\* | Lead Qualification \& Regional Sales Routing |

| \*\*Trigger\*\* | Manual Trigger (Mock Data) |

| \*\*Architecture\*\* | Enrichment → Scoring → Territory Routing |

| \*\*Programming\*\* | JavaScript (Code Nodes) |

| \*\*Integrations\*\* | HTTP APIs, Slack, Google Sheets (Optional) |

| \*\*Lead Categories\*\* | High • Medium • Low |

| \*\*Regions\*\* | US • EU |

| \*\*Notification Platform\*\* | Slack |

| \*\*Development Mode\*\* | Uses Mock Lead Data |



\---



\# 1. Overview



Not every inbound lead deserves the same amount of sales attention. Some organizations are ideal enterprise customers, while others may not yet meet the company's qualification criteria.



This workflow automatically analyzes incoming leads, enriches company information, calculates a qualification score, determines the lead's geographic region, and routes qualified opportunities to the appropriate sales team.



Instead of manually reviewing every lead, sales representatives only receive prospects that meet predefined business requirements.



Lower-priority leads are automatically placed into a nurture queue where they can continue receiving marketing engagement until they become sales-ready.



```

Manual Trigger

&#x20;     │

&#x20;     ▼

Mock Lead Data

&#x20;     │

&#x20;     ▼

Should Generate Email?

&#x20;     ├───────────────┐

&#x20;     │               │

&#x20;     ▼               ▼

Send to Nurture   Company Enrichment

Campaign                │

&#x20;                        ▼

&#x20;              Combine Enrichment Data

&#x20;                        │

&#x20;                        ▼

&#x20;                 Lead Scoring

&#x20;                        │

&#x20;                        ▼

&#x20;              Score Classification

&#x20;         ┌────────┬────────┬────────┐

&#x20;         ▼        ▼        ▼

&#x20;      High      Medium     Low

&#x20;         │         │         │

&#x20;         ▼         ▼         ▼

&#x20;   Region Router Region Router Nurture Queue

&#x20;     ┌────┴────┐

&#x20;     ▼         ▼

&#x20;    US         EU

&#x20;     │         │

&#x20;     ▼         ▼

Slack Notification

```



\---



\# 2. What Problem Does It Solve?



Sales organizations often spend significant time manually reviewing every incoming lead before deciding whether it should be assigned to a sales representative.



This manual qualification process creates several challenges:



\- Sales representatives waste time reviewing poor-quality leads.

\- High-value prospects are delayed while waiting for manual review.

\- Regional assignments become inconsistent.

\- Sales prioritization varies between team members.

\- Marketing and sales teams lack standardized qualification criteria.



This workflow standardizes lead evaluation by automatically enriching lead information, calculating qualification scores, assigning geographic ownership, and notifying the appropriate regional sales team.



\---



\# 3. Benefits



\- Eliminates manual lead qualification.

\- Standardizes lead scoring.

\- Automatically enriches company information.

\- Routes qualified leads by geographic region.

\- Prioritizes enterprise opportunities.

\- Sends automated Slack notifications.

\- Separates marketing-qualified and sales-qualified leads.

\- Improves response time.

\- Supports scalable lead processing.

\- Creates a repeatable sales qualification process.



\---



\# 4. Who Will Use It?



This workflow is ideal for organizations managing inbound sales pipelines.



Typical users include:



\- Sales Development Representatives (SDRs)

\- Revenue Operations Teams

\- Marketing Operations

\- Sales Managers

\- CRM Administrators

\- Business Development Teams

\- Enterprise Sales Teams



\---



\# 5. Workflow Breakdown



\## `When clicking "Execute workflow"` — Manual Trigger



Starts the workflow during development.



In production this node could be replaced with:



\- Website Form

\- CRM Trigger

\- Webhook

\- Marketing Automation Platform

\- API Endpoint



\---



\## `mockInfo` — Code



Generates mock lead information.



Typical data includes:



\- Company Name

\- Industry

\- Employee Count

\- Country

\- Revenue

\- Contact Information



This allows the workflow to be demonstrated without production systems.



\---



\## `Generate Email?` — Switch



Determines whether the lead is eligible for sales qualification.



\### False Branch



Leads that do not qualify immediately enter the nurture process.



These prospects are forwarded into marketing campaigns rather than sales outreach.



\---



\## `Sent to Nurture Sheet`



Stores low-priority leads for future nurturing.



This provides visibility into deferred opportunities while maintaining a clean sales pipeline.



\---



\## `Slack Message Update`



Optionally notifies internal teams that a lead has been placed into the nurture program.



\---



\# Lead Enrichment Pipeline



\## `Company Size` — HTTP Request



Retrieves company enrichment data.



Typical enrichment includes:



\- Employee Count

\- Revenue

\- Industry

\- Company Size

\- Headquarters

\- Organization Type



\---



\## `Status` — HTTP Request



Retrieves additional qualification information about the organization.



Examples include:



\- Active Status

\- Business Verification

\- Company Health

\- Operational Status



\---



\## `Enrichment Combination` — Merge



Combines enrichment responses into a single standardized lead profile.



This ensures downstream nodes work with a complete lead record regardless of data source.



\---



\## `Score` — Set



Calculates the lead qualification score.



Scoring may consider:



\- Company Size

\- Revenue

\- Industry

\- Growth Potential

\- Geographic Fit

\- Strategic Alignment



\---



\## `scorePath` — Switch



Routes leads into one of three qualification levels.



\### High Priority (Score: 100)



Enterprise-quality opportunities requiring immediate sales engagement.



\---



\### Medium Priority (Score: 50)



Qualified opportunities suitable for standard sales follow-up.



\---



\### Low Priority (Score: 10)



Leads that require additional nurturing before becoming sales-ready.



\---



\# High Priority Branch



\## `Score: 100`



Formats enterprise lead information.



\---



\## `US | EU`



Determines whether the enterprise lead belongs to the US or EU sales organization.



\---



\## `UScompressData`



Formats the payload for the US sales team.



\---



\## `EUcompressData`



Formats the payload for the European sales team.



\---



\## `InfoSentToSlack`



Sends enterprise lead notifications to Slack.



This enables immediate visibility for sales representatives.



\---



\# Medium Priority Branch



\## `Score: 50`



Formats mid-tier opportunities.



\---



\## `US | EU 1`



Routes qualified leads according to region.



\---



\## `UScompressData1`



Formats US sales notifications.



\---



\## `EUcompressData1`



Formats EU sales notifications.



\---



\## `InfoSentToSlack2`



Sends medium-priority sales alerts.



\---



\# Low Priority Branch



\## `Score: 10`



Formats low-priority lead information.



\---



\## `Nurture Queue`



Routes leads into long-term marketing engagement.



These prospects remain available for future campaigns instead of entering the active sales pipeline.



\---



\## `Slack Message`



Optionally informs internal teams that the lead has entered the nurture process.



\---



\# Error Handling



\## `Error Alert`



Captures failures occurring during company enrichment.



Instead of silently failing, the workflow creates an operational alert so administrators can investigate API or data issues.



\---



\# Setup Requirements



| Service | Used For | Credential Needed |

|----------|----------|------------------|

| Company Enrichment API | Company Intelligence | API Key |

| Business Status API | Company Verification | API Key |

| Slack API | Team Notifications | OAuth2 |

| Google Sheets (Optional) | Nurture Tracking | OAuth2 |



\---



\# Known Limitations / Next Steps



\- Mock enrichment APIs should be replaced with production services such as Clearbit, Apollo, or ZoomInfo.

\- Lead scoring currently uses predefined rules and could be enhanced with AI-powered predictive scoring.

\- Regional routing currently supports US and EU; additional territories can easily be added.

\- CRM synchronization is not included and could be added before sending Slack notifications.

\- Retry logic and centralized monitoring should be implemented for production deployments.



\---



\# Future Improvements



\- AI Lead Scoring

\- CRM Integration

\- Salesforce Synchronization

\- HubSpot Integration

\- Microsoft Teams Notifications

\- Duplicate Detection

\- Buying Intent Signals

\- AI Lead Summaries

\- Automated SDR Assignment

\- Sales Performance Dashboard

