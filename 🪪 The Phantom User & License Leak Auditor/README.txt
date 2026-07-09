# Automated Software License & Asset Reconciliation Engine

An enterprise-grade, code-free automation workflow built natively in n8n. This system serves as an automated auditor that cross-references a company's Human Resources Information System (HRIS) data against software seat exports to detect license leaks, calculate financial waste, and flag provisioning anomalies.

---

## 🚀 Overview

In large organizations, enterprise software licenses (e.g., Salesforce, Slack, GitHub) are frequently left active after an employee departs, resulting in significant financial waste ("phantom users"). Conversely, active employees are sometimes missed during provisioning, leading to operational bottlenecks.

This workflow automatically reconciles these systems without relying on external AI dependencies or custom JavaScript code, using pure native data engineering and conditional routing patterns within n8n.

---

## 🛠️ System Architecture & Workflow Logic

The engine processes data utilizing three foundational architectural design patterns:

1. **Source of Truth (SSOT) Alignment:** 
   Imports two separate datasets (the master HR active roster and the software seat export) and performs a full structural comparison.
   
2. **Content-Based Routing:** 
   Anomalies and discrepancies are instantly categorized using multi-branch conditional logic:
   - **Active but Unlicensed:** Identifies current employees missing necessary access and routes them to a provisioning queue.
   - **Inactive but Licensed (Phantom Users):** Isolates terminated accounts that are actively consuming premium licenses.

3. **Data Aggregation & Calculation:** 
   Flattens the stream of phantom users into a single evaluation metric, calculates total financial overhead waste dynamically using expression logic, and determines whether an emergency IT alert or low-priority log is required.

---

## 📊 Mock Data Schema

The workflow operates entirely out of a self-contained, sandbox-friendly environment using the following mock datasets:

### 1. Master HR Roster (Input 1)
Identifies the company's currently authorized personnel.
- Fields: `empId`, `name`, `status`, `dept`

### 2. Software Seat Export (Input 2)
Identifies active license accounts currently being billed (e.g., Salesforce Premium at $150/month per seat).
- Fields: `licenseId`, `userEmpId`, `software`

---

## 🔧 Core n8n Node Configurations (No-Code Blueprint)

- **Manual/Schedule Trigger:** Boots the workflow on demand or via a recurring cron-job.
- **Merge Node:** Configured to "Compare Two Datasets" mapping `empId` to `userEmpId` to isolate all non-matching data entries.
- **IF Node (Active User?):** Evaluates `status == 'Active'` on the isolated anomalies to separate provisioning issues from true financial leaks.
- **Edit Fields Node (totalSeats):** Combines the data using the native `{{ $input.all().length }}` evaluation method alongside the "Execute Once" parameter to deliver a single, consolidated math calculation object.
- **IF Node (Financial Threshold):** Runs a conditional test on the calculated value to dictate downstream communication priority.

---

## 🎯 Final Operational Output Example

When executed, the system distills hundreds of conflicting data lines down into a single, actionable compliance object:

[
  {
    "totalSeats": 2,
    "totalMonthlyWaste": 300
  }
]

This data is then natively dispatched to a high-priority IT alert webhook or a low-priority system log bucket depending on your threshold parameters.