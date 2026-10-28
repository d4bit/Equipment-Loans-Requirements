# Solution Design – Lumen Diagnostics Demo Equipment Loans

Source: project brief (equipment-loans-requirements.pdf v1.0). Org: SDO (production, Enterprise Edition, validate-only until release). API 62.0.

## 1. Architecture overview
A fully declarative Sales Cloud extension:
- **Data:** new private custom object `Equipment_Loan__c` with a lookup to the customer `Account`.
- **Automation:** before-save flow for rep/manager/owner (FT-103), validation rules for date policy (FT-103), approval process routed to `Sales_Manager__c` + Shipped/Approval Status guards (FT-104), schedule-triggered flow for return reminders (FT-106).
- **Email:** Classic Text templates in folder `Equipment_Loans`, sent through email alerts from an org-wide email address.
- **UI:** custom tab, record page with Status path, Account related list (FT-103).
- **Security:** Private OWD on the loan, owner = sales rep, permission sets for reps and managers (FT-108).
- **Reporting:** custom report type and "Open Equipment Loans by Equipment" report (US-117, waiting on PO).
- No integrations, no Apex.

| Capability | Feature / Stories | Status of refinement |
|---|---|---|
| Loan object & fields | FT-103 / US-109 | Specified; waiting on PO (Status lifecycle, equipment name) |
| Date validation | FT-103 / US-110 | Ready for Sprint |
| Rep/manager auto-population | FT-103 / US-111 | Ready for Sprint |
| Loan UI | FT-103 / US-112 | Specified; waiting on PO (Status values, sales app) |
| Manager approval | FT-104 / US-113 | Ready for Sprint |
| Shipped / Approval Status guard | FT-104 / US-114 | Ready for Sprint |
| Return reminder | FT-106 / US-116 | Ready for Sprint (sender address + send time pending PO, non-blocking for build) |
| Report | FT-107 / US-117 | Requires additional information |
| Permission sets / visibility | FT-108 / US-118, US-119 | Not yet refined |
| Spreadsheet migration | FT-105 / US-120 | Not yet refined |

## 2. Org facts (verified)
| Fact | Value | Impact |
|---|---|---|
| Account OWD | Public Read/Write (`DefaultAccountAccess = Edit`) | Master-detail sharing would expose all loans → lookup + Private OWD (ADR-01) |
| Person Accounts | Enabled (RTs `PersonAccount`, `SDO_PersonAccounts`) | Lookup filter restricts loans to business accounts |
| Business account record types | `Business_Account`, `SDO_Account_Simple`, `SDO_Account_Partner` | Account layouts to modify in US-112 |
| Sales apps | `LightningSales` (standard), `SDO_Sales_App` (custom) | Target app pending PO |
| Users without manager | 38 active standard users have no `ManagerId` | Manager data must be fixed before approvals go live (TK-115) |
| `Equipment_Loan__c` | Does not exist | New object |
| Org default time zone | `America/Los_Angeles` | Reminder schedule time and "tomorrow" computed in Pacific time |
| Organization-wide email addresses | **None** | Sender address must be created and verified before deploying email alerts |
| Task automation | Only active trigger: SDO demo `SDO_Tool_SalesforceRewind_Task`; no Task flows | Regression check for reminder Task inserts |
| Task picklists | `Status`: Not Started, In Progress, Completed, On Hold; `TaskSubtype` includes `Email`; `Priority`: High, Normal | Values used by US-116 |

## 3. Data model
### Equipment_Loan__c (new) — Private OWD, Grant Access Using Hierarchies, Name auto-number `EL-{00000}`, history tracking on, Allow Activities on
| Field | Type | Req. | Notes |
|---|---|---|---|
| `Name` | Auto Number `EL-{00000}` | – | Label "Loan Number" |
| `OwnerId` | Lookup(User/Queue) standard | – | Always = `Sales_Rep__c` (US-111 flow) |
| `Account__c` | Lookup(Account) | Yes | Relationship `Equipment_Loans`; delete Restrict; filter `IsPersonAccount = false`; reparenting blocked by VR |
| `Equipment_Name__c` | Text(255) | Yes | Pending: may become restricted picklist (GVS `Equipment_Name`) |
| `Serial_Number__c` | Text(80) | Yes | Not unique |
| `Start_Date__c` | Date | Yes | |
| `Return_Date__c` | Date | Yes | ≤ Start + 30 (inclusive, A3) |
| `Status__c` | Picklist restricted | Yes | `Requested` (default), `Shipped`; end-of-loan values pending |
| `Approval_Status__c` | Picklist restricted | No | `Not_Submitted` (default), `Pending`, `Approved`, `Rejected`; written only by the approval process |
| `Sales_Rep__c` | Lookup(User) | No | `Equipment_Loans_As_Rep`; active standard users only; reminder recipient |
| `Sales_Manager__c` | Lookup(User) | No | `Equipment_Loans_As_Manager`; = rep's `ManagerId`; approver |

Relationships: Account 1—n Equipment_Loan__c (lookup, restrict delete); User 1—n Equipment_Loan__c (owner, rep, manager); Equipment_Loan__c 1—n Task (reminder activity, `WhatId`).
Volumes: low (estimated tens per month, unconfirmed). Spreadsheet migration in US-120 (to refine).

### Task (standard, existing) — used by US-116
Reminder activity: `WhatId` = loan, `OwnerId` = Sales Rep, `Subject` = "Email: Equipment loan {Name} is due back tomorrow", `Status` = `Completed`, `Priority` = `Normal`, `TaskSubtype` = `Email`, `ActivityDate` = run date. No new fields.

## 4. Automation inventory
### Equipment_Loan__c
| Order | Component | Type | Event | Entry condition | Story |
|---|---|---|---|---|---|
| 1 | `Equipment_Loan_Before_Save_Set_Rep_Manager` | Record-triggered flow, before save | Create/Update | `ISNEW() \|\| ISCHANGED(Sales_Rep__c) \|\| ISCHANGED(OwnerId)` | US-111 |
| 2 | `Account_Cannot_Change` | Validation rule | Update | `AND(NOT(ISNEW()), ISCHANGED(Account__c))` | US-109 |
| 2 | `Return_Date_After_Start_Date` | Validation rule | Create/Update | `Return_Date__c <= Start_Date__c` | US-110 |
| 2 | `Return_Date_Max_30_Days` | Validation rule | Create/Update | `Return_Date__c - Start_Date__c > 30` | US-110 |
| 2 | `Shipped_Requires_Approval` | Validation rule | Create/Update | Status Shipped (new or changed) AND Approval Status ≠ Approved; skipped with `Bypass_Equipment_Loan_Validation` | US-114 |
| 2 | `Approval_Status_System_Only` | Validation rule | Create/Update | Approval Status set on create (≠ Not_Submitted) or changed on update; skipped with bypass permission | US-114 |
| – | `Equipment_Loan_Manager_Approval` | Approval process (1 step, Related User `Sales_Manager__c`) | Manual submit | Status Requested AND Approval Status blank/Not_Submitted/Rejected AND Sales Manager filled | US-113 |
| – | `EL_Set_Approval_Pending/Approved/Rejected/Not_Submitted` | Workflow field updates (approval actions) | Approval events | – | US-113 |
| – | `EL_Loan_Approved_Alert`, `EL_Loan_Rejected_Alert` | Email alerts (approval actions) → `Sales_Rep__c` | Approval events | – | US-113 |
| – | `Equipment_Loan_Return_Reminder` | Schedule-triggered flow, daily 07:00 org TZ | Schedule | Start filter `Status__c = Shipped` AND `Sales_Rep__c` not null; Decision `Return_Date__c = $Flow.CurrentDate + 1` AND `Sales_Rep__r.IsActive` | US-116 |
| – | `EL_Return_Reminder_Alert` | Email alert → `Sales_Rep__c` (from org-wide address) | Called by reminder flow | – | US-116 |

Order of execution: before-save flow runs before validation rules, so rules evaluate the flow-populated values. Approval-process field updates do not re-run custom validation rules (basis of US-114); they re-fire the before-save flow, whose entry condition is false for them. The reminder flow performs no DML on the loan (only Task inserts), so it does not interact with loan automation. No Apex triggers on the object.

### Task
| Component | Type | Notes |
|---|---|---|
| `SDO_Tool_SalesforceRewind_Task` (SDO, existing) | Apex trigger | Fires on reminder Task inserts — regression check in US-116 AC 7 |

### Email templates (Classic Text, folder `Equipment_Loans`)
| Template | Used by | Story |
|---|---|---|
| `Equipment_Loan_Approval_Request` | Approval assignment email | US-113 |
| `Equipment_Loan_Approved` | `EL_Loan_Approved_Alert` | US-113 |
| `Equipment_Loan_Rejected` | `EL_Loan_Rejected_Alert` | US-113 |
| `Equipment_Loan_Return_Reminder` | `EL_Return_Reminder_Alert` | US-116 |

## 5. Security model
| Element | Design |
|---|---|
| OWD `Equipment_Loan__c` | Private / Private, hierarchies on |
| Record ownership | Owner = Sales Rep (synced by US-111) → reps see their own loans |
| Managers | View All on `Equipment_Loan__c` via `Equipment_Loan_Manager` permission set (FT-108); required for approvers |
| Account owners | Access to loans on accounts they own when rep ≠ account owner — mechanism to define in US-119; note manual shares are removed on owner change |
| Permission sets | `Equipment_Loan_User` (CRUD R/C/E, tab, FLS read on Sales Rep/Manager/Approval Status), `Equipment_Loan_Manager` (adds View All, edit Sales Rep/Manager) — FT-108; `Equipment_Loan_Data_Migration` (custom permission `Bypass_Equipment_Loan_Validation` only, migration window only) — US-114 |
| Approval submitters | All Internal Users; record visibility is the gate (US-113) |
| Scheduled automation | Reminder flow runs as Automated Process user (system context); reminder Tasks owned by the Sales Rep |
| FLS caveat | Field-level-required fields are not listed in permission sets |

## 6. Integration inventory
None (out of scope §4). Outbound email only (Salesforce email alerts from an org-wide address).

## 7. Key decisions (ADR)

| # | Decision | Reason | Alternatives |
|---|---|---|---|
| D1 | `Equipment_Loan__c` required **lookup** to Account with own OWD Private; owner = rep (supersedes master-detail) | Account OWD is Public Read/Write in SDO; Controlled by Parent would expose every loan to every rep | Master-detail + changing Account OWD (org-wide impact); Apex sharing |
| D2 | Explicit `Sales_Rep__c` / `Sales_Manager__c` lookups filled by before-save flow, Owner kept equal to Sales Rep | Needed for approval routing, reminders and owner-based sharing | Use Account Owner (wrong if non-owner creates loan) |
| D3 | Standard approval process, single step, approver = Related User `Sales_Manager__c` | Native locking, recall, history and email; no code | Flow-based approval / Orchestrator; "Manager of submitter" |
| D4 | Initial submitters = All Internal Users | Related User not available for submitters; record access already restricts who can submit | Public group; Record Creator only |
| D5 | Two validation rules for the approval gate (Shipped guard + Approval Status system-only) | Shipped guard alone can be bypassed by anyone with FLS edit on Approval Status | Apex trigger; FLS only |
| D6 | Bypass via custom permission `Bypass_Equipment_Loan_Validation` in a dedicated permission set; date rules have no bypass | Migration needs historical Shipped/Approved loans without deactivating rules; invalid dates must be fixed at source | Deactivating rules in production; profile-based bypass |
| D7 | Classic email templates for approval emails | `{!ApprovalRequest.Comments}` / approval link only in Classic templates | Lightning templates |
| D8 | Approval not reset when an approved loan is edited (assumption) | Not requested; keeps design simple | Reset via validation/flow — pending Lumen answer |
| D9 | Migration as one-off CLI/Python ETL with idempotent upsert on `Legacy_Loan_Id__c` (Text(100), unique) | Few hundred rows; re-runnable; no permanent importer metadata; key = row ID or serial + start date | Data Import Wizard (no upsert on custom external ID with lookups resolution/error codes); Flow/Apex importer |
| D10 | Migration resolves reps by `User.Username` and accounts by `AccountNumber` (name fallback), outputting record Ids | `User.Email` not unique in SDO; `AccountNumber` unique and populated; name matching only as fallback with ambiguity check | Email/name matching; `Account.External_ID__c` (meaning unknown to Lumen) |
| D11 | Custom report type `Equipment_Loans_with_Account` (primary loan, Account via lookup) for US-117 | Deployable, controlled field set, stable report references; possible because Account is a lookup | Auto-generated standard report types (not controllable, naming tied to object) |
| D12 | Report folder shared View with All Internal Users; filter "Status equals Requested, Shipped" | Managers not identifiable by role/group in SDO; data already protected by object perms + Private OWD/View All. Inclusion filter auto-excludes future end statuses | Folder per role/public group (needs group maintenance); "Status not equal" filter (breaks when values are added) |
| D13 | Run Reports granted through `Equipment_Loan_Manager` | SDO-Sales profile lacks Run Reports; access via permission sets, not profiles | Profile change; separate reporting permission set |

## 8. Risks and open points
| # | Risk / open point | Impact | Mitigation |
|---|---|---|---|
| R1 | Status lifecycle and definition of "open" unconfirmed | Path, list views, report | US-109/US-112 held; additive picklist change; reminder filter (`Shipped` only) unaffected |
| R2 | Equipment name free text vs controlled list | Report grouping quality, data model | Hold US-109 build until answered |
| R3 | 30-day rule inclusive assumed (A3) | Boundary behaviour | One-character change in `Return_Date_Max_30_Days` |
| R4 | 38 active users without manager | Approval routing fails | TK-115 before go-live |
| R5 | Shared SDO org: modifying Account layouts / Sales app | Other demos affected | Additive changes only; validate-only until release |
| R6 | PO may reject ADR-01 | Rework of data model/sharing | Confirm before US-109 build |
| R7 | Account-owner visibility when rep ≠ account owner | Visibility gap | Design in US-119 |
| R8 | Target sales app unknown | Tab placement | ask_human on US-112 |
| R9 | No org-wide email address in SDO; sender address unknown | Email alert deployment fails; emails from wrong sender | ask_human on US-116; pre-deploy step creates + verifies the address |
| R10 | Reminder send time (07:00 Pacific assumed) and `$Flow.CurrentDate` time zone of Automated Process user | Reminder on wrong day near midnight UTC | ask_human on US-116; TZ check in US-116 how-to-test |
| R11 | Manual re-run of the reminder flow on the same day | Duplicate reminder | Accepted; only the schedule runs it |
| R12 | US-113 tech spec text still mentions "Controlled by Parent / detail object has no owner" (pre-ADR-01 wording) | Confusion for developer; design itself is unaffected (submitters = All Internal Users) | Clean up wording when US-113 is picked up |
| R13 | Production deliverability must be All Email | No approval/reminder emails | Release checklist (US-113, US-116) |

## 4. Security model

| Element | Setting | Story |
|---|---|---|
| `Equipment_Loan__c` OWD | Private; owner = sales rep (synced with `Sales_Rep__c`) | US-109 / US-111 |
| Account OWD | Public Read/Write (verified) — therefore loans do not inherit account sharing | US-109 |
| Account-owner access when rep ≠ account owner | To be designed in US-119 (pending PO meaning of "own accounts") | US-119 |
| `Equipment_Loan_User` | Read/Create/Edit; FLS read-only on `Approval_Status__c`, `Sales_Rep__c`, `Sales_Manager__c`; no access to `Legacy_Loan_Id__c` | US-118 |
| `Equipment_Loan_Manager` | Read/Create/Edit + View All; FLS read-only on `Approval_Status__c`, `Legacy_Loan_Id__c`; edit on `Sales_Rep__c`; **system permission Run Reports** (SDO-Sales profile lacks it — verified) | US-118 / US-120 / US-117 |
| `Equipment_Loan_Data_Migration` | Custom permission `Bypass_Equipment_Loan_Validation` only; assigned to the migration admin user during the load window only | US-114 / US-120 |
| Approval submitters | All Internal Users (record visibility is the gate) | US-113 |
| Approver | Related User `Sales_Manager__c`; must hold `Equipment_Loan_Manager` (checked in TK-115) | US-113 / TK-115 |
| Edit while pending | Admin or current approver | US-113 |
| Report folder `Equipment_Loan_Reports` ("Equipment Loans") | Shared View with All Internal Users; report data limited by object Read (report type access) and record sharing | US-117 |

## 10. Reporting

| Component | Type | Details | Story |
|---|---|---|---|
| `Equipment_Loans_with_Account` | Custom report type | Primary `Equipment_Loan__c` only; Account (Name, AccountNumber, Owner), Sales Rep name and Sales Manager name added via lookup; category Accounts & Contacts | US-117 |
| `Equipment_Loan_Reports` | Report folder (label "Equipment Loans") | Shared View to All Internal Users; distinct API name from Classic email folder `Equipment_Loans` | US-117 |
| `Open_Equipment_Loans_by_Equipment` | Summary report | Scope All; filter `Status__c` equals `Requested,Shipped`; grouped by `Equipment_Name__c` (A→Z) with counts; columns Loan Number, Account Name, Return Date, Start Date, Serial Number, Status, Approval Status, Sales Rep; sort Return Date asc | US-117 |

Visibility: managers see all loans through View All; reps (if they have Run Reports) see only loans shared with them. Future end-of-loan statuses are excluded automatically by the filter; only a new *open* status requires a report change.

## 8. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Rep ≠ account owner and "own accounts" undefined | Account owner may not see loan | US-119 design after PO answer |
| Reps without manager / managers without `Equipment_Loan_Manager` | Submission refused or approver cannot open the record | TK-115 queries before go-live |
| `User.ManagerId` changes after loans are created | Approvals routed to old manager | Documented; admin re-sets Sales Rep to recalc |
| Approved loan edited before shipping | Manager approved different data | Open question (D8) |
| Status lifecycle beyond `Shipped` unknown | Path, list views, report "open" definition incomplete | Open question; picklist extensible; report filter is inclusion-based (D12) |
| Email deliverability not All Email in production | No approval/reminder emails | Post-deploy check in US-113/US-116 |
| Bypass permission left assigned after migration | Approval gate can be bypassed | US-120 AC 7 / V6: 0 assignments after load |
| Migration loaded after the daily reminder run | Loans due next day miss their reminder | Load before 07:00 or send V5 list to reps |
| Spreadsheet loans already extended beyond 30 days | Rows rejected (`DATE_RULE`) | Lumen corrects return date at source; no bypass |
| Account name matching in migration | Wrong or ambiguous account | Ask Lumen for account numbers; ambiguity → error file |
| Free-text Equipment Name | Near-duplicate groups in the US-117 report | Controlled list if PO chooses it (same API name, report unaffected) |
| Rejected loans never resubmitted stay `Requested` | Remain in "open" report indefinitely | `Cancelled` status pending lifecycle answer |
| Managers on SDO-Sales profile lack Run Reports | Cannot run the report | Run Reports in `Equipment_Loan_Manager` (D13) |
| Report column tokens for lookup fields in custom report types | Validation failure of report metadata | Build + retrieve in a dev/scratch org before validating against SDO |

## 9. Open questions

- Status lifecycle after `Shipped` and definition of "open" (US-117 assumes Requested + Shipped; alternative Shipped only).
- Should sales reps run the "Open Equipment Loans by Equipment" report (would need Run Reports for SDO-Sales reps)?
- Equipment name free text vs controlled list.
- Meaning of "own accounts" (owner vs account team) — impacts US-119.
- Spreadsheet migration in scope? If yes: volume, account identifier, rep usernames, are all "not shipped" rows unapproved, cut-over date (US-120).
- Rejection behaviour (A4), delegated approvers, re-approval after edits of an approved loan, inclusive 30 days (A3), reminder time (A5).
