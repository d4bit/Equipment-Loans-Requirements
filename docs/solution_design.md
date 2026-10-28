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
### ADR-01 — Lookup to Account with Private OWD instead of master-detail
- **Decision:** `Account__c` is a required lookup; the loan has its own Private OWD and an owner kept equal to the Sales Rep.
- **Reason:** Account OWD is Public Read/Write; Controlled-by-Parent would grant every rep read/edit on every loan, violating §3.
- **Alternatives:** (a) master-detail + change Account OWD to Private — org-wide regression in a shared SDO; (b) master-detail + restriction rules — more complex, still edit access.
- **Status:** Applied; PO confirmation requested (ask_human on US-109). Supersedes earlier master-detail decision.

### ADR-02 — Owner ↔ Sales Rep sync in a before-save flow
- **Decision:** One before-save flow defaults Sales Rep from Owner on create and keeps Owner and Sales Rep equal on change; computes Sales Manager from `User.ManagerId`.
- **Reason:** Private sharing needs the owner; the brief and approval/reminder logic use an explicit Sales Rep field. Before-save = no extra DML.
- **Alternatives:** Drop `Sales_Rep__c` and use Owner only; Apex trigger (unnecessary).

### ADR-03 — Validation rules for date policy
- **Decision:** Two validation rules (after-start, ≤ 30 days inclusive) on Return Date, no bypass.
- **Reason:** Simple same-record logic; precise messages.
- **Alternatives:** One combined rule (less precise message); flow/Apex (overkill).

### ADR-04 — Declarative only
- **Decision:** No Apex in the solution. **Reason:** brief NFR declarative-first; all needs met by flow/VR/approval process/metadata.

### ADR-05 — Approval gate with two validation rules
- **Decision:** `Shipped_Requires_Approval` + `Approval_Status_System_Only` (US-114), both skippable only via custom permission `Bypass_Equipment_Loan_Validation`.
- **Reason:** Rule 1 alone can be bypassed by anyone with edit FLS on Approval Status; approval field updates don't re-run validation rules, so the guard doesn't block the approval process.
- **Alternatives:** Read-only FLS only (admins/API bypass it); Apex trigger (unnecessary).

### ADR-06 — Classic Text email templates
- **Decision:** All loan emails use Classic Text templates in folder `Equipment_Loans`.
- **Reason:** `{!ApprovalRequest.Comments}` (rejection reason) and `{!Equipment_Loan__c.Link}` (record link) are Classic merge fields; lookup merge renders names; one template technology for the project.
- **Alternatives:** Lightning templates (no approval merge fields, record-link handling less reliable).

### ADR-07 — Return reminder: schedule-triggered flow + email alert + logged Task
- **Decision:** Daily schedule-triggered flow on Shipped loans; Decision checks Return Date = tomorrow and active rep; email alert to Sales Rep; Create Records Task (Subtype Email, Completed) on the loan.
- **Reason:** Run-time evaluation automatically honours date/status changes (no "sent" flag); email alerts don't log activities, so a Task gives the activity trail (AC 4); scheduled-flow start filters can't use TODAY()+1, hence the Decision; volumes tiny.
- **Alternatives:** Flow "Send Email" action with `logEmailOnSend` (template merge with User recipient + related record and absolute links unreliable in Automated Process context); time-based scheduled path on a record-triggered flow (re-scheduling on date change is fragile, and paths on past dates fire immediately); Batch Apex (overkill).

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
