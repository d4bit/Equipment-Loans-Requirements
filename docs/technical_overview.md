# Technical overview

## Equipment Loan (Equipment_Loan__c)

Introduced by US-109. 100% declarative.

**Object:** Name = Loan Number, auto-number `EL-{00000}`. OWD Private (internal/external), hierarchy access on. Reports, Activities, Field History, Search, Sharing, Bulk API, Streaming API enabled. No record types. Lookup (not master-detail) to Account because Account OWD is Public Read/Write.

| Field | Type | Required | Notes |
|---|---|---|---|
| `Account__c` | Lookup(Account) | Yes | Delete = Restrict; filter `IsPersonAccount = False`; relationship `Equipment_Loans` |
| `Equipment_Name__c` | Restricted picklist (GVS `Equipment_Name`) | Yes | API = label, contains spaces |
| `Serial_Number__c` | Text(80) | Yes | Not unique |
| `Start_Date__c` / `Return_Date__c` | Date | Yes | Range enforced by US-110 rules |
| `Status__c` | Restricted picklist | Yes | Requested (default), Approved, Shipped, Returned, Rejected, Cancelled; open = first three |
| `Approval_Status__c` | Restricted picklist | No | Not_Submitted (default), Pending, Approved, Rejected; approval-process only |
| `Sales_Rep__c` | Lookup(User) | No | Filter active Standard users |
| `Sales_Manager__c` | Lookup(User) | No | Approver (EP-102) |

History tracked: 9 custom fields + Owner (10/20).

**Validation rules** (all active, no bypass, apply to admins and API loads):

| Rule | Story | Formula | Error on | Message |
|---|---|---|---|---|
| `Account_Cannot_Change` | US-109 | `AND(NOT(ISNEW()), ISCHANGED(Account__c))` | | |
| `Return_Date_After_Start_Date` | US-110 | `AND(NOT(ISBLANK(Start_Date__c)), NOT(ISBLANK(Return_Date__c)), Return_Date__c <= Start_Date__c)` | `Return_Date__c` | Return Date must be after Start Date. |
| `Return_Date_Max_30_Days` | US-110 | `AND(NOT(ISBLANK(Start_Date__c)), NOT(ISBLANK(Return_Date__c)), Return_Date__c - Start_Date__c > 30)` | `Return_Date__c` | Return Date must be no more than 30 days after Start Date. |

The 30-day maximum is inclusive (PO confirmed). The two date rules are mutually exclusive. Apex test data must use Return Date = Start Date + 1..30. If migration is approved later, add a custom permission `Bypass_Equipment_Loan_Validation` to the formulas.

**Global value set:** `Equipment_Name`: Ultrasound U1, Patient Monitor M5, Blood Analyzer A200, Infusion Pump P3, ECG Device E12.

**Permissions:** see "Equipment Loan access control (US-118)". Field-level required fields must not be in permission set field permissions.

**Deployment:** no manual steps. Manifests are `docs/stories/US-109/manifest/US-109-package.xml` and `docs/stories/US-110/manifest/US-110-package.xml`. US-110 must deploy with or after US-109.

## Equipment Loan automation (US-111)

**Flow `Equipment_Loan_Before_Save_Set_Rep_Manager`:** before-save record-triggered flow on `Equipment_Loan__c`, runs on create and update, Active, API 62.0. No Apex.

| Item | Detail |
|---|---|
| Entry | `OR(ISNEW(), ISCHANGED(Sales_Rep__c), ISCHANGED(OwnerId))` |
| Rep from owner | Runs when Sales Rep is blank, or the owner changed and the rep did not: `Sales_Rep__c = OwnerId` |
| Owner from rep | Runs on a new loan with a rep given, or on a rep change: `OwnerId = Sales_Rep__c`. The rep wins if both change. |
| Manager | Get Records on User (`Id = Sales_Rep__c`) → `Sales_Manager__c = ManagerId`. Blank if the rep has no manager. |
| Fault | Custom error "The Sales Manager could not be determined from the Sales Rep…" |
| Performance | 1 SOQL per batch of 200, no DML. Runs in system context, so reps don't need Transfer Record. |

- Edits that change neither the rep nor the owner do not recalculate the manager.
- Changing a user's manager does not update existing loans.
- Queues are not enabled on the object, so the owner is always a User.

**Deployment:** ships with US-109 in the same release. Manifests are `docs/stories/US-111/manifest/US-111-package.xml` and `US-111-with-US-109-validate-package.xml` (combined check-only validation). Before go-live, fill in `User.ManagerId` for every sales rep. If the flow was deployed inactive, activate it.

## Equipment Loan access control (US-118)

Updated by US-119: loan visibility comes from loan ownership only. Managers are read-only.

| Component | Type | Settings |
|---|---|---|
| `Equipment_Loan__c` sharing | OWD | Internal Private, external Private; Grant Access Using Hierarchies on (US-109) |
| `Equipment_Loan_User` | Permission set (no license) | Equipment_Loan__c Read/Create/Edit; Account Read and Contact Read; `Approval_Status__c`, `Sales_Rep__c` and `Sales_Manager__c` Read only |
| `Equipment_Loan_Manager` | Permission set (no license) | Equipment_Loan__c Read/Create + View All, **no Edit** (US-119); `Sales_Rep__c` and `Sales_Manager__c` Read + Edit (effective only on create); `Approval_Status__c` Read only |
| `Equipment_Loan_Managers` | Public group | Unused since US-119 (cleanup candidate) |
| Sharing rules | – | None. `Equipment_Loan_Managers_Edit` was removed in US-119 |

| Who | Access |
|---|---|
| Owner (= Sales Rep, set by the US-111 flow) | Read/Edit, no Delete |
| Managers | Read all (View All). They approve without Edit |
| Owner's role-hierarchy superiors | Same as the owner, if they have a perm set that grants Edit |
| Admins | Full (only they reassign and delete loans) |

- There is no account-owner sharing, no `Equipment_Loan__Share` logic and no account-transfer automation.
- No business user has Delete or Modify All.
- Contact Read is required alongside Account Read because Person Accounts are enabled.
- **Post-deploy:** assign the permission sets from Lumen's lists of reps and managers. Managers get `Equipment_Loan_Manager` only. No group membership is needed. In any org that already has `Equipment_Loan_Managers_Edit`, deploy `docs/stories/US-119/manifest/destructiveChangesPost.xml`.

## Equipment Loan approval guards (US-114)

| Component | Type | Behaviour |
|---|---|---|
| `Shipped_Requires_Approval` | Validation rule (error on `Status__c`) | Blocks `Status__c` = `Shipped` on insert or when Status changes, unless `Approval_Status__c` = `Approved` |
| `Approval_Status_System_Only` | Validation rule (error on `Approval_Status__c`) | On insert, allows only a blank or `Not_Submitted` Approval Status. On update, blocks any change to Approval Status |
| `Bypass_Equipment_Loan_Validation` | Custom permission | Both rules check `NOT($Permission.Bypass_Equipment_Loan_Validation)` |
| `Equipment_Loan_Data_Migration` | Permission set | Grants only the custom permission. Unassigned by default; US-120 is cancelled |

Notes:
- Approval process field updates (US-113) don't re-run validation rules, so the approval actions are not blocked. Verify this in UAT.
- The US-110 date rules deliberately ignore the bypass permission.
- No Apex is involved.

## User manager prerequisite (TK-115)

TK-115 was cancelled by the PO. The project doesn't maintain User.ManagerId; the admin does that outside the project. Sales_Manager__c is set from the Sales Rep's ManagerId only when a loan is created or its Sales Rep changes (US-111). If the rep has no manager, the field stays empty and the approval can't route. Changing a user's manager doesn't recalculate existing loans: an admin re-saves the loan's Sales Rep. Only admins may delete loans (the US-118 permission sets have allowDelete=false).

## Equipment Loan user interface (US-112)

| Component | API name | Notes |
|---|---|---|
| Tab | `Equipment_Loan__c` | Not in any app. Visible via `tabSettings` in `Equipment_Loan_User` and `Equipment_Loan_Manager` |
| Layout | `Equipment_Loan__c-Equipment Loan Layout` | Sections Loan Details / Approval (`Approval_Status__c` Readonly) / System Information. Excludes `ChangeOwnerOne` and `ChangeRecordType`. Actions: Edit, Submit, Delete. Related lists: `RelatedProcessHistoryList`, `RelatedHistoryList`, `RelatedEntityHistoryList` |
| Compact layout | `Equipment_Loan_Compact` | Name, Account__c, Equipment_Name__c, Status__c, Return_Date__c; assigned on the object |
| FlexiPage | `Equipment_Loan_Record_Page` | Desktop default via the View `actionOverrides` on the object |
| Path | `Equipment_Loan_Status_Path` | `Status__c`, all values, active (needs Path Settings enabled) |
| List views | `Open_Loans`, `My_Loans` | Open_Loans: scope Everything, Status in Requested/Approved/Shipped. My_Loans: scope Mine |
| Account related list | `Equipment_Loans` (on `Account__c`) | **Manual post-deploy step** on the `Business_Account`, `SDO_Account_Simple` and `SDO_Account_Partner` layouts. Not in metadata, so the shared SDO layouts are not overwritten |

Rep read-only access to Sales Rep and Sales Manager comes from FLS (US-118), not from the layout.

## Equipment Loan return reminder (US-116)

| Component | Details |
|---|---|
| Flow `Equipment_Loan_Return_Reminder` | Schedule-triggered. Runs daily at 08:00 America/Los_Angeles as the Automated Process user. Start filter: `Status__c` = `Shipped` AND `Sales_Rep__c` not null. |
| Decision `Is_Reminder_Due` | `Return_Date__c` = `$Flow.CurrentDate` + 1 AND `Sales_Rep__r.IsActive`. |
| Email alert `EL_Return_Reminder_Alert` | Recipient: user lookup `Sales_Rep__c`. Sender: `DefaultWorkflowUser`. Template: `Equipment_Loans/Equipment_Loan_Return_Reminder` (Classic Text, uses `{!Equipment_Loan__c.Link}`). |
| Task `Log_Reminder_Activity` | `WhatId` = loan, `OwnerId` = Sales Rep, Subtype `Email`, Status `Completed`, `ActivityDate` = run date. |
| Email folder `Equipment_Loans` | Public, ReadOnly. Shared with US-113. |

**Error handling and side effects**
- The flow has no fault paths on purpose. Errors go to the flow error email recipient.
- The flow does not update `Equipment_Loan__c`.
- The Task insert fires the SDO managed trigger `SDO_Tool_SalesforceRewind_Task`.

**Deployment**
- `workflows/Equipment_Loan__c.workflow-meta.xml` is shared with US-113. Merge the alerts and field updates from both stories.
- Post-deploy: set the Default Workflow User, check that Scheduled Jobs shows 08:00 Pacific, and set Deliverability to All Email.
- There is no Apex and no permission set change.

## Equipment Loan approval process (US-113)

Declarative, no Apex.

| Item | Value |
|---|---|
| Approval process | `Equipment_Loan__c.Equipment_Loan_Manager_Approval` (active, order 1) |
| Entry criteria | `Status__c` = `Requested`; `Approval_Status__c` blank, `Not_Submitted` or `Rejected`; `Sales_Manager__c` not blank |
| Submitters | All internal users (record visibility is the gate) |
| Step | `Manager_Approval`, approver = `Sales_Manager__c`, delegates not allowed |
| Lock | Locked while pending (admin or current approver can edit); unlocked after approval, rejection and recall |
| Recall | Allowed |

| Phase | Field update (`Approval_Status__c`) | Alert |
|---|---|---|
| Submit | `EL_Set_Approval_Pending` → `Pending` | Assignment email `Equipment_Loan_Approval_Request` |
| Approve | `EL_Set_Approval_Approved` → `Approved` | `EL_Loan_Approved_Alert` → Sales Rep |
| Reject | `EL_Set_Approval_Rejected` → `Rejected` | `EL_Loan_Rejected_Alert` → Sales Rep (with comments) |
| Recall | `EL_Set_Approval_Not_Submitted` → `Not_Submitted` | – |

- Email templates: Classic Text in folder `Equipment_Loans`. Alerts use sender CurrentUser because the org has no org-wide address.
- Field updates run in system context and do not re-run validation rules, which US-114 relies on. Flow `Equipment_Loan_Before_Save_Set_Rep_Manager` re-fires but has no effect.
- Shared file `workflows/Equipment_Loan__c.workflow-meta.xml` also contains US-116's `EL_Return_Reminder_Alert`. Always edit it starting from the merged content.
- Post-deploy: check that the process is Active, set Deliverability = All Email, and make sure reps have a User.ManagerId.
- Open point: unlock after approval vs. the PO's "approved is locked" (#28).

## Equipment Loan reporting (US-117)

| Component | Details |
|---|---|
| ReportType `Equipment_Loans_with_Account` | Primary object `Equipment_Loan__c` only; Account fields come through the `Account__c` lookup (column token `Equipment_Loan__c$Account__c.Name`). Category `accounts`. |
| Report folder `Equipment_Loan_Reports` ("Equipment Loans") | Shared View with All Internal Users (`AllInternalUsers` / `Organization`). Data is protected by object permissions and the Private OWD. |
| Report `Open_Equipment_Loans_by_Equipment` | Summary report grouped by `Equipment_Name__c` Asc, sorted by `Return_Date__c` Asc. Filter: `Status__c` = Requested, Approved, Shipped. Date range: all time. Shows subtotals and a grand total. |
| `Equipment_Loan_Manager` | Grants `RunReports` (US-117). `Equipment_Loan_User` does not grant it, and rep profiles are unchanged (PO decision). |

There is no Apex or automation. If a new "open" status is added, update the report filter.
