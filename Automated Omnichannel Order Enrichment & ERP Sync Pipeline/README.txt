📦 Project Title: Automated Omnichannel Order Enrichment & ERP Sync Pipeline

🚀 Overview

An automated backend data pipeline built in n8n designed to ingest, process, enrich, and route regional e-commerce and enterprise B2B orders. The system normalizes incoming payloads, injects localized regulatory/routing compliance details, performs a bulk database transactional sync, and implements real-time error-handling architecture.

---

🛠️ Architecture Flow

* Data Ingestion: Webhook entry parsing real-time order streams.
* Financial & Logic Engine: JavaScript transformation layers scaling subtotals, tax matrices, and routing criteria.
* Regional Localization Core: Tri-branch routing architecture dividing data into tailored regions:
`US-EAST-LAX` (USD tracking)
`EU-WEST-AMS` (EUR + GDPR protection compliance tagging)
`APAC-SOUTH-SIN` (SGD + customs duty pre-clearance tracking)


* ERP Database Sync: Consolidated array flattening for high-efficiency, single-payload database writes.
* Error Gatekeeping: Conditional response inspection (`id: 101` validation) separating functional traffic from server fallbacks.

---

⚙️ Configuration & Environment Settings

Node 17: Database Write (HTTP Request)

* Method: `POST`
* Body Content Type: `JSON`
* Expression Payload: `{{ $input.all().map(item => item.json) }}`
* Error Resilience Setting: `Continue (detaching error info)` / `Never Fail` enabled to capture network state drops gracefully.

Node 18: Traffic Control Error Gate (If Node)

* Data Type Comparison: `String`
* Condition: `{{ $json.id }}` $\rightarrow$ **Equals** $\rightarrow$ `101`
* Routing Logic:
* `True` $\rightarrow$ Routes directly to Node 20 (Client Handshake Success Response).
* `False` $\rightarrow$ Routes to Node 19 (DevOps Slack Notification / Alert Handler).



### Node 20: Webhook Response Handshake

* Execution Mode: `Run Once for All Items`
* Output Payload Structure:**
```json
{
  "status": "success",
  "statusCode": 200,
  "message": "Omnichannel order pipeline executed successfully.",
  "timestamp": "YYYY-MM-DDTHH:MM:SS.MMMZ",
  "database_receipt": {
    "id": 101,
    "status_text": "Records Bulk-Written to Warehouse"
  }
}

```



---

📈 Key Operations & Monitoring

* Deployment: Flip the *Active* toggle switch in the top-right corner to point live production webhooks toward this workflow endpoint.
* Error Recovery: If an outage is detected on the ERP end, check the tracking executions stemming from the *False* branch of Node 18 to isolate missing indices.