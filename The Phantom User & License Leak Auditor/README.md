# IT License Audit & User Compliance Workflow

An automated n8n workflow that compares an organization's HR roster against software license assignments to detect inactive accounts, identify users consuming licenses without employment records (phantom users), verify license utilization, and generate IT alerts for corrective action.

---

## Quick Facts

| Item | Value |
|------|------|
| **Workflow Type** | IT Operations Automation |
| **Primary Purpose** | License Compliance & User Audit |
| **Trigger** | Manual Trigger (Mock Data) |
| **Architecture** | Data Reconciliation & Compliance Validation |
| **Programming** | JavaScript (Code Nodes) |
| **Integrations** | HR System, Identity Provider, Messaging Platform |
| **Audit Focus** | Active Users • Phantom Users • License Utilization |
| **Output** | IT Notifications & Compliance Reports |
| **Development Mode** | Uses Mock HR and License Data |

---

# 1. Overview

Organizations often purchase more software licenses than they actually need. Over time, employees leave the company while their accounts remain active, consuming valuable licenses and creating unnecessary security risks.

This workflow automates the license audit process by comparing the organization's master HR roster with the current list of licensed users.

Users that exist in the licensing system but no longer appear in HR are identified as **phantom users**. Active employees are evaluated separately to determine whether they require a valid software license.

When licensing capacity exceeds acceptable thresholds or orphaned accounts are detected, the workflow automatically generates an IT alert for administrative review.

The result is a repeatable compliance process that helps reduce software costs while improving security and identity governance.

```
Manual Trigger
      │
      ▼
Retrieve HR Roster
      │
Retrieve Licensed Users
      │
      ▼
Compare User Lists
      │
      ▼
Remove Phantom Users
      │
      ▼
Is User Active?
      ├───────────────┐
      │               │
      ▼               ▼
Active User     Inactive User
      │               │
Check License     Notify User
Utilization
      │
      ▼
License Threshold
      ├───────────────┐
      │               │
      ▼               ▼
Generate IT Alert   Log Result
```

---

# 2. What Problem Does It Solve?

Software licenses represent a significant operational expense for many organizations.

Without regular audits:

- Former employees may continue consuming paid licenses.
- Inactive accounts increase security risks.
- IT teams manually compare HR exports against licensing reports.
- License utilization becomes difficult to monitor.
- Organizations may purchase unnecessary additional licenses.

This workflow automates the reconciliation process by identifying orphaned accounts, validating active users, and notifying IT whenever corrective action is required.

---

# 3. Benefits

- Reduces unnecessary software licensing costs.
- Detects phantom user accounts.
- Improves identity governance.
- Supports software compliance audits.
- Reduces manual reconciliation work.
- Improves IT security.
- Generates actionable IT alerts.
- Standardizes license review processes.
- Supports scalable periodic audits.
- Creates a repeatable compliance workflow.

---

# 4. Who Will Use It?

This workflow is designed for organizations responsible for managing employee identities and software licensing.

Typical users include:

- IT Administrators
- Identity & Access Management (IAM) Teams
- HR Operations
- Software Asset Management Teams
- Compliance Officers
- Security Teams
- Internal Audit Teams

---

# 5. Workflow Breakdown

## `When clicking "Execute workflow"` — Manual Trigger

Starts the workflow during development.

In production this node could be replaced with:

- Scheduled Daily Audit
- Weekly Cron Job
- HR Webhook
- Identity Provider Trigger

---

## `Master HR Roster` — Code

Retrieves the organization's current employee roster.

Typical fields include:

- Employee ID
- Full Name
- Department
- Employment Status
- Business Unit
- Email Address

This represents the authoritative source of active employees.

---

## `The Audit Target` — Code

Retrieves the list of users currently consuming software licenses.

Depending on the organization, this data may originate from:

- Microsoft 365
- Google Workspace
- Okta
- Azure Active Directory
- SaaS Administration APIs

---

## `Filter Phantom Users` — Merge

Compares both datasets and identifies licensing discrepancies.

This node detects:

- Users with licenses but no HR record.
- Duplicate identities.
- Accounts requiring further review.

Only valid employee records continue into the compliance pipeline.

---

## `Active User?` — Switch

Determines whether the employee is currently active.

### Active Branch

Continues into license utilization analysis.

### Inactive Branch

Routes inactive users into notification handling.

---

## `Message (Active w/o License)`

Generates a notification indicating that an active employee does not currently possess the required software license.

This enables IT administrators to provision licenses proactively.

---

## `totalSeats` — Set

Calculates current license utilization.

Typical metrics include:

- Total Purchased Licenses
- Assigned Licenses
- Available Licenses
- Remaining Capacity
- Utilization Percentage

These values are used for compliance validation.

---

## `If` — Conditional

Evaluates whether license usage exceeds predefined thresholds.

Possible checks include:

- Less than 10% licenses remaining.
- License pool exhausted.
- Unexpected utilization spike.
- Capacity planning threshold reached.

---

## `IT Alert`

Generates a notification for IT administrators.

Alerts may include:

- Additional license purchases required.
- Phantom account removal.
- License reassignment.
- Capacity warnings.

---

## `low-priority log`

Records informational audit results that do not require immediate action.

Examples include:

- Normal utilization.
- Successful reconciliation.
- Informational audit summaries.

---

# Setup Requirements

| Service | Used For | Credential Needed |
|----------|----------|------------------|
| HR Information System | Employee Roster | API Credentials |
| Identity Provider | User Accounts | OAuth2/API Token |
| Messaging Platform | IT Notifications | API Credentials |
| Software License API | License Utilization | API Credentials |

---

# Known Limitations / Next Steps

- HR and licensing datasets currently use mock data for demonstration purposes.
- User matching is based on simplified identifiers and should include unique employee IDs in production.
- Threshold values are static and could be replaced with configurable business rules.
- Notification delivery is represented generically and can be integrated with Slack, Microsoft Teams, or email.
- Historical audit results should be stored for long-term reporting and compliance tracking.

---

# Future Improvements

- Microsoft 365 Integration
- Google Workspace Integration
- Okta Integration
- Azure Active Directory Integration
- Automated License Reclamation
- Scheduled Compliance Reports
- Executive Dashboard
- Historical Audit Analytics
- AI-Powered License Forecasting
- Multi-Tenant Support
