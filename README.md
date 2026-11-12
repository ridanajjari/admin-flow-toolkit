# Admin Flow Toolkit (Salesforce)

Five Salesforce Flows for common admin automation, packaged as source-format metadata with the
custom fields, custom label and permission set they need.

I did not write the flow XML by hand. I deployed each flow to an org and retrieved it with
`sf project retrieve start`, so the committed files are what the platform emits.

---

## Why I built it

Most Salesforce orgs need the same basic automations: welcome emails, approval reminders, duplicate
prevention, auto-close logic, inactivity alerts. I built them here as declarative Flows (no Apex).

---

## What's included

| #   | Flow API name                         | Type                                                   | Trigger                             | What it does                                                                      |
| --- | ------------------------------------- | ------------------------------------------------------ | ----------------------------------- | --------------------------------------------------------------------------------- |
| 1   | `Account_Welcome_Email_On_Creation`   | Record-Triggered (After Save, Create)                  | New Account                         | Emails the Account Owner a summary of the new Account                             |
| 2   | `Opportunity_Approval_Reminder`       | Schedule-Triggered (Opportunity)                       | Daily 08:00 UTC                     | Emails owners of open Opportunities pending approval for 48h or more              |
| 3   | `Contact_Duplicate_Check_Before_Save` | Record-Triggered (Before Save, Create)                 | New Contact with an email           | Blocks the save if another Contact has the same email                             |
| 4   | `Case_Auto_Close_After_Resolution`    | Record-Triggered (After Save, Update) + Scheduled Path | Case updated to Status = `Resolved` | 72h later, closes the Case if it is still `Resolved`                              |
| 5   | `Lead_Inactivity_Alert_To_Owner`      | Schedule-Triggered (Lead)                              | Daily 09:00 UTC                     | Emails owners of open Leads with no activity for 7+ days, at most once per 7 days |

All flows ship as `<status>Draft</status>`. Review them, then activate them in Setup → Flows.

---

## How each flow works

### Flow 1: Account welcome email on creation

- **Type:** Record-Triggered, After Save, Create only.
- **Sends:** Account name, industry, phone and website to `$Record.Owner.Email`.

### Flow 2: Opportunity pending approval reminder

- **Type:** Schedule-Triggered, daily. The start element selects Opportunities with
  `Approval_Status__c = Pending`, `Submitted_For_Approval_Date__c` not blank and `IsClosed = false`.
  The flow then runs once per matching record (`$Record`), so there is no Get Records or Loop.
- **48h logic:** a Decision compares `Submitted_For_Approval_Date__c` with the formula
  `NOW() - 2` (48 hours ago). I put the check in the Decision because schedule-trigger filters
  cannot express "now minus 48 hours".
- **Repeats:** the Opportunity owner gets a reminder every day until the approval status changes.

### Flow 3: Contact duplicate email check before save

- **Type:** Record-Triggered, Before Save, Create only. Entry criteria: `Email` is not blank.
- **Logic:** Get Records on Contact by exact email, then a custom field error on `Email` if one exists.
- **Scope:** insert only, exact (case-insensitive SOQL) email match. I don't check email changes on
  update.

#### Flow 3 vs standard Duplicate Rules

Salesforce already has Matching Rules and Duplicate Rules (Setup → Duplicate Management), and a
"Standard Contact Duplicate Rule" is active in many orgs. I would use Duplicate Rules when you need
fuzzy matching (name + email, phone, address), "allow with alert" instead of block, duplicate
reporting, or checks on insert **and** update without building anything. A flow like this one only
makes sense for a narrow rule that Duplicate Rules cannot express, or when you need custom logic
around the match (for example a specific error message or extra conditions). If you run both on the
same field, users get two different errors for the same problem, so pick one.

### Flow 4: Case auto-close 72h after resolution

- **Type:** Record-Triggered, After Save, Update, with "only when a record is updated to meet the
  condition requirements" (`doesRequireRecordChangedToMeetCriteria`) and entry criteria
  `Status = Resolved`.
- **Logic:** a Scheduled Path runs 72 hours after the triggering update. It checks again that
  `Status` is still `Resolved`, then sets `Status = Closed`. (My earlier version used a Wait
  element, which record-triggered flows do not support.)
- **Prerequisite:** `Resolved` is **not** a standard Case Status value. Add it under
  Setup → Object Manager → Case → Fields → Status before you activate the flow (not marked as closed).

### Flow 5: Lead inactivity alert after 7 days

- **Type:** Schedule-Triggered, daily. The start element selects Leads with `IsConverted = false`
  and `Status != Closed - Not Converted`.
- **Logic:** a Decision sends the alert only when
  - the last activity date is 7+ days ago, using `CreatedDate` when `LastActivityDate` is blank
    (I treat Leads that never had an activity as inactive since creation), and
  - `Last_Inactivity_Alert__c` is blank or 7+ days ago, and
  - the Lead owner is a user (I skip queue-owned Leads).
- After sending the email, the flow stamps `Last_Inactivity_Alert__c` with today's date, so an owner
  gets at most one alert per Lead every 7 days instead of one every day.

### Error handling (fault paths)

Every Get Records, Update Records and Send Email element has a fault connector:

- Flows 1, 2, 4 and 5 send faults to a `Notify_Admin_Of_Fault` Send Email action. It emails the
  address stored in the custom label `Flow_Fault_Notification_Email` with the flow name, record Id
  and `$Flow.FaultMessage`.
- Flow 3 is Before Save, where actions are not available, so it blocks the save with a custom error
  showing `$Flow.FaultMessage`.

The notification action itself has no fault connector. If it fails, the standard Salesforce flow
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
| Case Status value `Resolved`                                                            | Flow 4     | No, add it manually                        |
| Email deliverability set to "All email" (Setup → Deliverability)                        | 1, 2, 4, 5 | No, org setting                            |

Something has to populate `Approval_Status__c` and `Submitted_For_Approval_Date__c` (an approval
process field update, another flow, or users). The toolkit only reads them.

---

## Setup

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

- On 12 Nov 2026 I deployed everything to a Base Edition trial org at API 68.0 (all five flows, the
  three custom fields, the custom label and the permission set) and retrieved it again. The
  committed XML is the retrieved XML.
- On 12 Nov 2026 I also deployed the flows to a Developer Edition org (API 68.0). The XML I
  retrieved from it was identical to what is committed.
- In the trial org I activated all five flows for a short time (activation runs the full flow
  validation), then deactivated them again.
- I smoke-tested Flows 1 and 3 in the trial org by creating records through the API: a new Account
  saved without error, a second Contact with the same email was blocked with the flow's error
  message, and a Contact without an email saved normally. I deleted the test records afterwards.
- Not tested yet: Flows 2 and 5 (scheduled) and Flow 4 (72h scheduled path; the trial org has no
  `Resolved` status) were only validated by activation. I have not exercised their runtime
  behaviour or real email delivery yet.

---

## Salesforce admin concepts used

- **Before Save flow:** validation without DML, custom field error
- **After Save flow:** actions after the record is committed
- **Schedule-triggered flow with object + filters:** one interview per matching record, no loop
- **Scheduled Path:** time-based follow-up in a record-triggered flow (replaces Wait elements)
- **Run only when the record is updated to meet the criteria**
- **Fault paths:** every data and email element handles failures
- **Formula resources:** `NOW() - 2`, blank-date fallback with `ISBLANK` / `DATEVALUE`

---

## Author

**Rida Najjari**, Salesforce Consultant, Developer & Administrator

4x Salesforce Certified: Platform Administrator · Platform App Builder · Platform Developer I · Associate

- LinkedIn: [linkedin.com/in/rida-najjari](https://www.linkedin.com/in/rida-najjari)
- GitHub: [github.com/ridanajjari](https://github.com/ridanajjari)
