# Admin Flow Toolkit — Salesforce

A collection of 5 production-ready Salesforce Flows covering the most common admin automation scenarios. Each Flow is packaged as metadata and can be deployed to any org in minutes.

---

## Business Problem

Every Salesforce org needs the same basic automations: welcome emails, approval reminders, duplicate prevention, auto-close logic, inactivity alerts. Admins rebuild these from scratch every time. This toolkit gives you ready-to-deploy Flows with zero code required.

---

## Flows Included

| # | Flow Name | Type | Trigger | What It Does |
|---|-----------|------|---------|--------------|
| 1 | Account — Welcome Email on Creation | Record-Triggered (After Save) | New Account created | Emails the Account Owner with a welcome summary |
| 2 | Opportunity — Pending Approval Reminder | Scheduled | Daily at 8am | Reminds owners of Opportunities pending approval > 48h |
| 3 | Contact — Duplicate Email Check Before Save | Record-Triggered (Before Save) | New Contact created | Blocks save if another Contact with same email exists |
| 4 | Case — Auto-Close 72h After Resolution | Record-Triggered (After Save) | Case set to Resolved | Automatically closes the Case after 72 hours if not reopened |
| 5 | Lead — Inactivity Alert After 7 Days | Scheduled | Daily at 9am | Alerts sales reps about Leads with no activity in 7 days |

---

## Flow Details

### Flow 1 — Account Welcome Email on Creation
- **Type:** Record-Triggered, After Save, Insert only
- **What it sends:** Account name, industry, phone, website to the owner
- **Use case:** New business onboarding, partner accounts

### Flow 2 — Opportunity Pending Approval Reminder
- **Type:** Scheduled, runs every day at 8am
- **Logic:** Queries all Opportunities where `Approval_Status__c = 'Pending'` and `Submitted_For_Approval_Date__c` is older than 48h
- **Prerequisite:** Custom fields `Approval_Status__c` and `Submitted_For_Approval_Date__c` on Opportunity

### Flow 3 — Contact Duplicate Email Check Before Save
- **Type:** Record-Triggered, Before Save, Insert only
- **Logic:** Queries existing Contacts by email → shows inline field error if found
- **Why Before Save:** Cheaper than After Save, no record is written if blocked

### Flow 4 — Case Auto-Close 72h After Resolution
- **Type:** Record-Triggered, After Save, Update
- **Logic:** Starts a 72-hour wait when Status = "Resolved". After 72h, checks if still Resolved → sets to Closed
- **Why useful:** Reduces agent workload, enforces SLA closure

### Flow 5 — Lead Inactivity Alert After 7 Days
- **Type:** Scheduled, runs every day at 9am
- **Logic:** Queries open, unconverted Leads with `LastActivityDate <= TODAY()-7` → loops and sends email per Lead
- **Why useful:** Prevents pipeline rot and cold leads

---

## Deployment

```bash
git clone https://github.com/YOUR_USERNAME/admin-flow-toolkit.git
cd admin-flow-toolkit

# Option 1 — Deploy via SFDX
sf org login web --alias my-org
sf project deploy start --source-dir flows --target-org my-org

# Option 2 — Manual import
# Go to Setup → Flows → New Flow → Import from File
# Upload the .flow-meta.xml file you want
```

### Prerequisites by Flow

| Flow | Custom Fields Needed |
|------|---------------------|
| Flow 1 | None |
| Flow 2 | `Opportunity.Approval_Status__c` (Picklist), `Opportunity.Submitted_For_Approval_Date__c` (DateTime) |
| Flow 3 | None |
| Flow 4 | None |
| Flow 5 | None |

---

## Key Salesforce Admin Concepts Demonstrated

- **Before Save Flow** — field validation without DML, blocks record creation inline
- **After Save Flow** — triggers actions after a record is committed
- **Scheduled Flow** — runs on a cron-like schedule, queries and loops records
- **Wait element** — time-based pause inside a Flow (auto-close after 72h)
- **Custom Error element** — displays a user-friendly inline validation message
- **Loop element** — iterates over a record collection inside a Flow
- **Formula in Flow** — `TODAY() - 7` for dynamic date filters

---

## Author

**Rida Najjari** — Salesforce Consultant, Developer & Administrator

4x Salesforce Certified: Platform Administrator · Platform App Builder · Platform Developer I · Associate

- LinkedIn: [linkedin.com/in/rida-najjari](https://www.linkedin.com/in/rida-najjari)
- GitHub: [github.com/ridanajjari](https://github.com/ridanajjari)
