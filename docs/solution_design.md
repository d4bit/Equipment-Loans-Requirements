# Solution Design – Lumen Diagnostics Demo Equipment Loans

Source: project brief (equipment-loans-requirements.pdf v1.0). Org: SDO (production, Enterprise Edition, validate-only until release). API 62.0.

## 1. Architecture overview

A fully declarative Sales Cloud extension:
- **Data:** new private custom object `Equipment_Loan__c` with a lookup to the customer `Account`.
- **Automation:** before-save flow for rep/manager/owner (FT-103), validation rules for date policy (FT-103), approval process routed to `Sales_Manager__c` + Shipped/Approval Status guards (FT-104), schedule-triggered flow for return reminders (FT-106), after-save flows that maintain account-owner sharing (FT-108).
- **Email:** Classic Text templates in folder `Equipment_Loans`, sent through email alerts from an org-wide email address.
- **UI:** custom tab, record page with Status path, Account related list (FT-103).
- **Security:** Private OWD on the loan, owner = sales rep, flow-maintained read share for the account owner, permission sets for reps and managers, manager edit via public group + owner-based sharing rule (FT-108).
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
| Permission sets / manager access | FT-108 / US-118 | Ready for Sprint (user lists needed only for post-deploy assignment) |
| Loan visibility (account-owner sharing) | FT-108 / US-119 | Ready for Sprint (3 non-blocking PO assumptions) |
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
| Permission sets `Equipment_Loan*`, public groups `Equipment*` | None exist | No name clash for US-114/US-118 |
| Account Teams | Enabled, **0** `AccountTeamMember` rows | "Own accounts" = Account Owner only (A7); no team-based loan sharing |
| Role hierarchy | Populated (CEO → VP of Sales → RVP East/Central/West → regional sales roles…); 12 active users without role | Hierarchy access on loans only reaches users who also hold a loan permission set |
| Account automation | Active after-update Apex triggers `SDO_Tool_SalesforceRewind_Account`, `LogAccountChange`; no active Account flows | Regression check for US-119 Flow 2 |
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
| 3 | `Equipment_Loan_After_Save_Share_Account_Owner` | Record-triggered flow, after save | Create/Update | `ISNEW() \|\| ISCHANGED(OwnerId)`; acts when `Account__r.OwnerId ≠ OwnerId` and account owner active → creates `Equipment_Loan__Share` (Read, Manual) if missing | US-119 |
| – | `Equipment_Loan_Manager_Approval` | Approval process (1 step, Related User `Sales_Manager__c`) | Manual submit | Status Requested AND Approval Status blank/Not_Submitted/Rejected AND Sales Manager filled | US-113 |
| – | `EL_Set_Approval_Pending/Approved/Rejected/Not_Submitted` | Workflow field updates (approval actions) | Approval events | – | US-113 |
| – | `EL_Loan_Approved_Alert`, `EL_Loan_Rejected_Alert` | Email alerts (approval actions) → `Sales_Rep__c` | Approval events | – | US-113 |
| – | `Equipment_Loan_Return_Reminder` | Schedule-triggered flow, daily 07:00 org TZ | Schedule | Start filter `Status__c = Shipped` AND `Sales_Rep__c` not null; Decision `Return_Date__c = $Flow.CurrentDate + 1` AND `Sales_Rep__r.IsActive` | US-116 |
| – | `EL_Return_Reminder_Alert` | Email alert → `Sales_Rep__c` (from org-wide address) | Called by reminder flow | – | US-116 |
| – | `Equipment_Loan_Managers_Edit` | Owner-based sharing rule (All Internal Users → group `Equipment_Loan_Managers`, Read/Write) | Sharing recalculation | – | US-118 |

Order of execution: the before-save flow runs before the validation rules, so the rules evaluate the values the flow filled in. Approval-process field updates do not re-run custom validation rules (this is what US-114 relies on). They do re-fire the before-save flow and the after-save sharing flow, but both entry conditions are false for them. On an owner change the platform deletes manual shares as part of the save, and the after-save sharing flow then re-creates the account-owner share. The reminder flow performs no DML on the loan (Task inserts only), so it does not interact with loan automation. There are no Apex triggers on the object.

### Account
| Component | Type | Event | Entry condition | Story |
|---|---|---|---|---|
| `SDO_Tool_SalesforceRewind_Account`, `LogAccountChange` (SDO, existing) | Apex triggers (after update) | – | – | existing |
| `Account_After_Save_Reshare_Equipment_Loans` | Record-triggered flow, after save | Update | `ISCHANGED(OwnerId)`; gets the account's loans, deletes the previous owner's Manual shares, creates Read/Manual shares for the new owner on loans not owned by them | US-119 |

The flow runs after the existing Apex after-triggers and writes only `Equipment_Loan__Share`. It ends straight after one query for accounts without loans.

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

### Record access on `Equipment_Loan__c`
| Who | Mechanism | Access | Story |
|---|---|---|---|
| Everyone (baseline) | OWD Private / Private | None | US-109 |
| Sales Rep | Record owner (`OwnerId` = `Sales_Rep__c`, synced by before-save flow) | Full (owner) | US-109 / US-111 |
| Account owner (when ≠ Sales Rep) | `Equipment_Loan__Share` RowCause `Manual`, maintained by `Equipment_Loan_After_Save_Share_Account_Owner` + `Account_After_Save_Reshare_Equipment_Loans` | Read | US-119 |
| Managers | `View All` in `Equipment_Loan_Manager` (read, independent of owner; required for approvers) + owner-based sharing rule `Equipment_Loan_Managers_Edit` to public group `Equipment_Loan_Managers` | Read/Write | US-118 |
| Role-hierarchy superiors of the owner | Grant Access Using Hierarchies = on | As owner (only effective with a loan permission set) | US-109 |
| Admins | Modify All Data | Full | – |

Reps therefore see (a) loans they are Sales Rep of and (b) loans on accounts they own, and nothing else, even though every Account is visible to them. The Account OWD is not changed.

### Permission sets
| API Name | Equipment_Loan__c | Account | Tab | FLS (non-required fields) | Custom permission | Story |
|---|---|---|---|---|---|---|
| `Equipment_Loan_User` | Read, Create, Edit | Read | Visible | `Approval_Status__c`, `Sales_Rep__c`, `Sales_Manager__c` = Read | – | US-118 |
| `Equipment_Loan_Manager` | Read, Create, Edit, View All | Read | Visible | `Approval_Status__c` = Read; `Sales_Rep__c`, `Sales_Manager__c` = Read/Edit | – | US-118 |
| `Equipment_Loan_Data_Migration` | – | – | – | – | `Bypass_Equipment_Loan_Validation` (migration window only) | US-114 |

No business user has Delete or Modify All on loans. Field-level-required fields are not listed in permission sets (deploy constraint); they are readable and editable for everyone with object access. Managers receive only `Equipment_Loan_Manager` (a superset of the user set) and membership of `Equipment_Loan_Managers`. Both are assigned post-deploy from Lumen's lists and checked with SOQL.

### Other
| Element | Design |
|---|---|
| Approval submitters | All Internal Users; record visibility is the gate (US-113) |
| Approvers | Must hold `Equipment_Loan_Manager` (View All) |
| Record-triggered / scheduled flows | System context; reps need no rights on `Equipment_Loan__Share`. Reminder flow runs as the Automated Process user; reminder Tasks are owned by the Sales Rep |
| Manual sharing by users | Sharing button not placed on layouts (US-112). Flow-maintained Manual shares can't be told apart from user-created ones |
| Owner change by managers | Done by the before-save flow (system context), so no Transfer/Modify All is needed. Verified in US-118 AC 3 |

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

### ADR-08 — Account-owner access via flow-maintained manual shares
- **Decision:** Two after-save record-triggered flows maintain an `Equipment_Loan__Share` row (Read, RowCause `Manual`) for the Account Owner when the owner is not the Sales Rep. One flow is on the loan (create / owner change), the other on Account (owner change).
- **Reason:** Sharing rules cannot target "the owner of the related account". Changing the Account OWD is ruled out (ADR-01). Flows are declarative (ADR-04) and volumes are tiny.
- **Alternatives:** (a) Apex managed sharing with a custom sharing reason, which survives owner changes but is code and out of scope; (b) forcing Sales Rep = Account Owner, which conflicts with A1 (manager can reassign the rep); (c) no account-owner access, which fails §3 when a colleague creates the loan.
- **Consequences:** The platform deletes manual shares on loan owner change, so the loan flow re-creates them. User-created manual shares for a previous account owner are also removed. The Sharing button is kept off layouts.

### ADR-09 — Manager edit via public group + owner-based sharing rule
- **Decision:** `Equipment_Loan_Manager` grants View All (read). Edit on all loans comes from sharing rule `Equipment_Loan_Managers_Edit` (records owned by All Internal Users → group `Equipment_Loan_Managers`, Read/Write).
- **Reason:** View All is read-only. Modify All would also grant delete and transfer, which the business rules exclude.
- **Alternatives:** Modify All (too broad); role hierarchy only (managers are not reliably above their reps; 12 users have no role); read-only managers (contradicts US-118 AC 3).
- **Consequences:** Group membership must mirror permission set assignment. This is a runbook step with a SOQL check.

## 8. Risks and open points

| # | Risk / open point | Impact | Mitigation |
|---|---|---|---|
| R1 | Status lifecycle and definition of "open" unconfirmed | Path, list views, report | US-109/US-112 held; additive picklist change; reminder filter (`Shipped` only) unaffected |
| R2 | Equipment name free text vs controlled list | Report grouping quality, data model | Hold US-109 build until answered |
| R3 | 30-day rule inclusive assumed (A3) | Boundary behaviour | One-character change in `Return_Date_Max_30_Days` |
| R4 | 38 active users without manager | Approval routing fails | TK-115 before go-live |
| R5 | Shared SDO org: modifying Account layouts / Sales app / Account automation | Other demos affected | Additive changes only; US-119 Account flow is entry-gated on owner change and exits immediately for accounts without loans; validate-only until release |
| R6 | PO may reject ADR-01 | Rework of data model/sharing | Confirm before US-109 build |
| R7 | Account-owner visibility when rep ≠ account owner | Visibility gap | **Resolved by design in US-119 (ADR-08)**; 3 assumptions with PO: Read access only, no loan reassignment on account transfer, reps may create loans on accounts they don't own |
| R8 | Target sales app unknown | Tab placement | ask_human on US-112 |
| R9 | No org-wide email address in SDO; sender address unknown | Email alert deployment fails; emails from wrong sender | ask_human on US-116; pre-deploy step creates + verifies the address |
| R10 | Reminder send time (07:00 Pacific assumed) and `$Flow.CurrentDate` time zone of Automated Process user | Reminder on wrong day near midnight UTC | ask_human on US-116; TZ check in US-116 how-to-test |
| R11 | Manual re-run of the reminder flow on the same day | Duplicate reminder | Accepted; only the schedule runs it |
| R12 | US-113 tech spec text still mentions "Controlled by Parent / detail object has no owner" (pre-ADR-01 wording) | Confusion for developer; design itself is unaffected (submitters = All Internal Users) | Clean up wording when US-113 is picked up |
| R13 | Production deliverability must be All Email | No approval/reminder emails | Release checklist (US-113, US-116) |
| R14 | Manual shares are deleted on loan owner change; flow must re-create them | Account owner loses access after a rep change | Flow 1 entry `ISCHANGED(OwnerId)`; US-119 AC 5 |
| R15 | `Equipment_Loan_Managers` group membership drifts from `Equipment_Loan_Manager` assignments | Manager can see but not edit, or vice versa | US-118 runbook SOQL check; admin procedure: always do both |
| R16 | Loans loaded before US-119 is deployed get no account-owner share | Visibility gap for migrated loans | Deploy US-119 before US-120 load; fallback Data Loader share insert |
| R17 | Owner change made by the before-save flow when a manager (non-owner) changes Sales Rep | Save error if the platform enforces transfer rights | Expected OK (system context); verified in US-118 AC 3, and escalated to the architect if it fails (no Modify All) |
| R18 | Role hierarchy on: owner's superiors with a loan permission set see/edit the owner's loans | A rep with a role above other reps would see their loans | Accepted (superiors = management); revisit if Lumen's reps are nested in the hierarchy |

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
