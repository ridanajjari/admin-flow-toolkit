# Admin Flow Toolkit — Salesforce

Five Salesforce Flows for common admin automation scenarios, packaged as source-format metadata
together with the custom fields, custom label and permission set they need.

The flow XML in this repo is what the platform emits: each flow was deployed to an org and then
retrieved with `sf project retrieve start`, so the committed files are not hand-written XML.

---

## Business Problem

Most Salesforce orgs need the same basic automations: welcome emails, approval reminders, duplicate
prevention, auto-close logic, inactivity alerts. This toolkit provides them as declarative Flows
(no Apex).

---

## Flows Included

| #   | Flow API name                         | Type                                                   | Trigger                             | What it does                                                                      |
| --- | ------------------------------------- | ------------------------------------------------------ | ----------------------------------- | --------------------------------------------------------------------------------- |
| 1   | `Account_Welcome_Email_On_Creation`   | Record-Triggered (After Save, Create)                  | New Account                         | Emails the Account Owner a summary of the new Account                             |
| 2   | `Opportunity_Approval_Reminder`       | Schedule-Triggered (Opportunity)                       | Daily 08:00 UTC                     | Emails owners of open Opportunities pending approval for 48h or more              |
| 3   | `Contact_Duplicate_Check_Before_Save` | Record-Triggered (Before Save, Create)                 | New Contact with an email           | Blocks the save if another Contact has the same email                             |
| 4   | `Case_Auto_Close_After_Resolution`    | Record-Triggered (After Save, Update) + Scheduled Path | Case updated to Status = `Resolved` | 72h later, closes the Case if it is still `Resolved`                              |
| 5   | `Lead_Inactivity_Alert_To_Owner`      | Schedule-Triggered (Lead)                              | Daily 09:00 UTC                     | Emails owners of open Leads with no activity for 7+ days, at most once per 7 days |

All flows ship with `<status>Draft</status>`. Review them, then activate them in Setup → Flows.

---

## Flow Details

### Flow 1 — Account Welcome Email on Creation

- **Type:** Record-Triggered, After Save, Create only.
- **Sends:** Account name, industry, phone and website to `$Record.Owner.Email`.

### Flow 2 — Opportunity Pending Approval Reminder

- **Type:** Schedule-Triggered, daily. The start element selects Opportunities with
  `Approval_Status__c = Pending`, `Submitted_For_Approval_Date__c` not blank and `IsClosed = false`;
  the flow then runs once per matching record (`$Record`), so there is no Get Records or Loop.
- **48h logic:** a Decision compares `Submitted_For_Approval_Date__c` with the formula
  `NOW() - 2` (48 hours ago). Schedule-trigger filters cannot express "now minus 48 hours", which is
  why the check lives in the Decision.
- **Repeats:** the owner is reminded every day until the approval status changes.

### Flow 3 — Contact Duplicate Email Check Before Save

- **Type:** Record-Triggered, Before Save, Create only; entry criteria `Email` is not blank.
- **Logic:** Get Records on Contact by exact email → custom field error on `Email` if one exists.
- **Scope:** insert only, exact (case-insensitive SOQL) email match; email changes on update are not
  checked.

#### Flow 3 vs standard Duplicate Rules

Salesforce ships Matching Rules and Duplicate Rules (Setup → Duplicate Management), and a
"Standard Contact Duplicate Rule" is active in many orgs. Prefer Duplicate Rules when you need fuzzy
matching (name + email, phone, address), "allow with alert" instead of block, duplicate reporting,
or checks on insert **and** update without building anything. Use a flow like this one only for a
narrow rule that Duplicate Rules cannot express, or when you need custom logic around the match
(for example, a specific error message or extra conditions). Running both on the same field gives
users two different errors for the same problem, so pick one.

### Flow 4 — Case Auto-Close 72h After Resolution

- **Type:** Record-Triggered, After Save, Update, with "only when a record is updated to meet the
  condition requirements" (`doesRequireRecordChangedToMeetCriteria`) and entry criteria
  `Status = Resolved`.
- **Logic:** a Scheduled Path runs 72 hours after the triggering update. It re-checks that
  `Status` is still `Resolved` and then sets `Status = Closed`. (The earlier version used a Wait
  element, which record-triggered flows do not support.)
- **Prerequisite:** `Resolved` is **not** a standard Case Status value. Add it under
  Setup → Object Manager → Case → Fields → Status before activating (not marked as closed).

### Flow 5 — Lead Inactivity Alert After 7 Days

- **Type:** Schedule-Triggered, daily. The start element selects Leads with `IsConverted = false`
  and `Status != Closed - Not Converted`.
- **Logic:** a Decision sends the alert only when
  - the last activity date is 7+ days ago, using `CreatedDate` when `LastActivityDate` is blank
    (Leads that never had an activity are treated as inactive since creation), and
  - `Last_Inactivity_Alert__c` is blank or 7+ days ago, and
  - the owner is a user (queue-owned Leads are skipped).
- After emailing, the flow stamps `Last_Inactivity_Alert__c` with today's date, so an owner gets at
  most one alert per Lead every 7 days instead of one every day.

### Error handling (fault paths)

Every Get Records, Update Records and Send Email element has a fault connector:

- Flows 1, 2, 4 and 5 route faults to a `Notify_Admin_Of_Fault` Send Email action that emails the
  address stored in the custom label `Flow_Fault_Notification_Email` with the flow name, record Id
  and `$Flow.FaultMessage`.
- Flow 3 (Before Save, where actions are not available) blocks the save with a custom error showing
  `$Flow.FaultMessage`.

The notification action itself has no fault connector; if it fails, Salesforce's standard flow
error email to the admin applies.

---

## Prerequisites

| Item                                                                                    | Needed by  | In this repo                               |
| --------------------------------------------------------------------------------------- | ---------- | ------------------------------------------ |
| `Opportunity.Approval_Status__c` (Picklist: Not Submitted, Pending, Approved, Rejected) | Flow 2     | Yes                                        |
| `Opportunity.Submitted_For_Approval_Date__c` (DateTime)                                 | Flow 2     | Yes                                        |
| `Lead.Last_Inactivity_Alert__c` (Date)                                                  | Flow 5     | Yes                                        |
| Custom label `Flow_Fault_Notification_Email`                                            | All        | Yes (value `admin@example.com`; change it) |
| Permission set `Admin_Flow_Toolkit` (FLS on the three fields)                           | Users      | Yes                                        |
| Case Status value `Resolved`                                                            | Flow 4     | No — add it manually                       |
| Email deliverability set to "All email" (Setup → Deliverability)                        | 1, 2, 4, 5 | No — org setting                           |

Note: something must populate `Approval_Status__c` and `Submitted_For_Approval_Date__c` (an
approval process field update, another flow, or users); this toolkit only reads them.

---

## Deployment

```bash
git clone https://github.com/ridanajjari/admin-flow-toolkit.git
cd admin-flow-toolkit

sf org login web --alias my-org
sf project deploy start --source-dir force-app --target-org my-org
sf org assign permset --name Admin_Flow_Toolkit --target-org my-org
```

Then:

1. Setup → Custom Labels → `Flow_Fault_Notification_Email` → set your admin address.
2. Add the `Resolved` Case Status value if you plan to use Flow 4.
3. Open each flow in Flow Builder, review it, and activate it.

### Local tooling

```bash
npm install
npm run lint             # ESLint (the repo has no Aura/LWC JavaScript yet, so this is a no-op)
npm run prettier:verify  # Prettier check; metadata XML is excluded so it stays as retrieved
```

---

## Testing

- ✅ Deployed and retrieved in a Salesforce org at API 68.0 (a Base Edition trial org, 2026-11-12):
  all five flows, the three custom fields, the custom label and the permission set. The committed
  XML is the retrieved XML.
- ✅ All five flows were temporarily activated in that org (activation runs the full flow
  validation), then deactivated again.
- ✅ Smoke test of Flows 1 and 3 by creating records through the API: a new Account saved
  without error; a second Contact with the same email was blocked with the flow's error message; a
  Contact without an email saved normally. Test records were deleted afterwards.
- ⏳ Flows 2 and 5 (scheduled) and Flow 4 (72h scheduled path; the org has no `Resolved` status)
  were validated by activation only; their runtime behaviour and actual email delivery have not been
  exercised yet.

---

## Key Salesforce Admin Concepts Demonstrated

- **Before Save flow** — validation without DML, custom field error
- **After Save flow** — actions after the record is committed
- **Schedule-triggered flow with object + filters** — one interview per matching record, no loop
- **Scheduled Path** — time-based follow-up in a record-triggered flow (replaces Wait elements)
- **Run only when the record is updated to meet the criteria**
- **Fault paths** — every data and email element handles failures
- **Formula resources** — `NOW() - 2`, blank-date fallback with `ISBLANK` / `DATEVALUE`

---

## Author

**Rida Najjari** — Salesforce Consultant, Developer & Administrator

4x Salesforce Certified: Platform Administrator · Platform App Builder · Platform Developer I · Associate

- LinkedIn: [linkedin.com/in/rida-najjari](https://www.linkedin.com/in/rida-najjari)
- GitHub: [github.com/ridanajjari](https://github.com/ridanajjari)
