# Solution Design – Lumen Diagnostics Demo Equipment Loans

Source: project brief (equipment-loans-requirements.pdf v1.0). Target org: SDO (production, validate-only until release). API 62.0.

## 1. Architecture overview
Declarative solution in the existing Sales Cloud org; no Apex, no integrations. The only non-metadata deliverable is a one-off migration toolkit (CLI + Python scripts) for US-120, if migration is confirmed.

| Layer | Components | Stories |
|---|---|---|
| Data | Custom object `Equipment_Loan__c`, required **lookup** to Account, own OWD Private | US-109 |
| UI | Tab, layout, compact layout, Lightning record page, Path, list views, Account related list | US-112 |
| Validation | Date rules, Account cannot change, Shipped guard, Approval Status guard | US-109, US-110, US-114 |
| Automation | Before-save flow (rep/owner/manager), approval process, scheduled reminder flow | US-111, US-113, US-116 |
| Reporting | Report "Open Equipment Loans by Equipment" | US-117 |
| Security | Permission sets, Private OWD + owner = rep, manager View All, account-owner access, bypass custom permission | US-118, US-119, US-114 |
| Data | User manager clean-up, optional spreadsheet migration | TK-115, US-120 |

Loan lifecycle (confirmed part): `Requested` → (manager approval) → `Shipped` → *end status pending confirmation*.

## 2. Data model
### Equipment_Loan__c (new)
Object: OWD Private (internal/external), Grant Access Using Hierarchies on, history tracking, reports, activities, search, Bulk API on.

| Field | Type | Req. | Notes |
|---|---|---|---|
| `Name` | Auto Number `EL-{00000}` | — | Label "Loan Number" |
| `Account__c` | Lookup(Account) | Yes | Relationship `Equipment_Loans`; delete Restrict; lookup filter business accounts only; no reparenting (validation rule) |
| `Equipment_Name__c` | Text(255) | Yes | May become restricted picklist (open question) |
| `Serial_Number__c` | Text(80) | Yes | Not unique |
| `Start_Date__c` | Date | Yes | |
| `Return_Date__c` | Date | Yes | |
| `Status__c` | Picklist (restricted) | Yes | `Requested` (default), `Shipped`, + pending values |
| `Approval_Status__c` | Picklist (restricted) | No | `Not_Submitted` (default), `Pending`, `Approved`, `Rejected`; system-maintained |
| `Sales_Rep__c` | Lookup(User) | No | Relationship `Equipment_Loans_As_Rep`; active standard users; = OwnerId; reminder recipient |
| `Sales_Manager__c` | Lookup(User) | No | Relationship `Equipment_Loans_As_Manager`; = rep's ManagerId; approver |
| `Legacy_Loan_Id__c` | Text(100), External ID, Unique (case-insensitive) | No | Spreadsheet key `SS-{rowId}` or `SS-{SERIAL}-{YYYYMMDD}`; only created if US-120 confirmed |

### Standard objects used
| Object | Usage |
|---|---|
| Account | Parent (lookup) of loans. OWD Public Read/Write in SDO (verified); Person Accounts enabled — loans restricted to business accounts. Migration match key `AccountNumber` (unique, populated on 48/52 business accounts) |
| User | `ManagerId` → `Sales_Manager__c`. Migration identifies reps by `Username` — `Email` is not unique in SDO (30 active users share `noreply@example.com`) |

## 3. Automation inventory
### Equipment_Loan__c
| Type | API Name | Event / Schedule | Entry condition | Story |
|---|---|---|---|---|
| Validation rule | `Account_Cannot_Change` | Save | Update and Account changed | US-109 |
| Validation rule | `Return_Date_After_Start_Date` | Save | Return ≤ Start (no bypass) | US-110 |
| Validation rule | `Return_Date_Max_30_Days` | Save | Return − Start > 30 (no bypass) | US-110 |
| Validation rule | `Shipped_Requires_Approval` | Save | Status → `Shipped` (new or changed) and Approval Status ≠ `Approved`, no bypass | US-114 |
| Validation rule | `Approval_Status_System_Only` | Save | Approval Status set on create (≠ blank/`Not_Submitted`) or changed on update, no bypass | US-114 |
| Record-triggered flow (before save) | `Equipment_Loan_Before_Save_Set_Rep_Manager` | Create/Update | `ISNEW() \|\| ISCHANGED(Sales_Rep__c) \|\| ISCHANGED(OwnerId)` — syncs Owner ⇄ Sales Rep, sets Sales Manager | US-111 |
| Approval process | `Equipment_Loan_Manager_Approval` | Manual submit | Status `Requested`, Approval Status blank/`Not_Submitted`/`Rejected`, Sales Manager not blank | US-113 |
| Field updates | `EL_Set_Approval_Pending/Approved/Rejected/Not_Submitted` | Approval actions | — | US-113 |
| Email alerts | `EL_Loan_Approved_Alert`, `EL_Loan_Rejected_Alert` | Final approval / rejection | Recipient `Sales_Rep__c` | US-113 |
| Schedule-triggered flow | `Equipment_Loan_Return_Reminder` | Daily 07:00 (to confirm) | Status `Shipped`, Return Date = today + 1, Sales Rep set & active | US-116 |
| Email alert | `EL_Return_Reminder_Alert` | From reminder flow | Recipient `Sales_Rep__c` | US-116 |

### Order of execution notes
- Approval field updates re-fire the before-save flow (no effect: entry condition false) and do **not** re-run custom validation rules — this is what allows `Approval_Status_System_Only` to block manual edits while the approval process still updates the field.
- Before-save flow changing OwnerId runs in system context; manual shares are dropped on owner change (relevant to US-119).
- Migration load (US-120): validation rules run, US-114 rules skipped via bypass custom permission, before-save flow sets Owner/Manager; no approval or email automation fires during the load.
- No Apex triggers on `Equipment_Loan__c`; if one is ever added, document its interaction with the before-save flow.

### Email templates
| Template | Kind | Folder | Story |
|---|---|---|---|
| `Equipment_Loan_Approval_Request`, `Equipment_Loan_Approved`, `Equipment_Loan_Rejected` | Classic Text (needs `{!ApprovalRequest.*}`) | `Equipment_Loans` | US-113 |
| `Equipment_Loan_Return_Reminder` | Lightning | Equipment Loans | US-116 |

## 4. Security model
| Element | Setting | Story |
|---|---|---|
| `Equipment_Loan__c` OWD | Private; owner = sales rep (synced with `Sales_Rep__c`) | US-109 / US-111 |
| Account OWD | Public Read/Write (verified) — therefore loans do not inherit account sharing | US-109 |
| Account-owner access when rep ≠ account owner | To be designed in US-119 (pending PO meaning of "own accounts") | US-119 |
| `Equipment_Loan_User` | Read/Create/Edit; FLS read-only on `Approval_Status__c`, `Sales_Rep__c`, `Sales_Manager__c`; no access to `Legacy_Loan_Id__c` | US-118 |
| `Equipment_Loan_Manager` | Read/Create/Edit + View All; FLS read-only on `Approval_Status__c`, `Legacy_Loan_Id__c`; edit on `Sales_Rep__c` | US-118 / US-120 |
| `Equipment_Loan_Data_Migration` | Custom permission `Bypass_Equipment_Loan_Validation` only; assigned to the migration admin user during the load window only | US-114 / US-120 |
| Approval submitters | All Internal Users (record visibility is the gate) | US-113 |
| Approver | Related User `Sales_Manager__c`; must hold `Equipment_Loan_Manager` (checked in TK-115) | US-113 / TK-115 |
| Edit while pending | Admin or current approver | US-113 |

## 5. Integration inventory
None (out of scope §4). Outbound email only (approval and reminder alerts); production deliverability must be All Email. The US-120 migration is a one-off file load (Bulk API 2.0 via `sf data upsert bulk`), not an integration.

## 6. Data & volumes
- Expected volume low (demo loans of one sales team); all automation is bulk-safe for 200 records/transaction.
- TK-115: populate `User.ManagerId` for reps before go-live (38 active standard users currently lack a manager).
- US-120 (pending confirmation): Lumen fills an intake CSV template (account number/name, equipment, serial, dates, shipped Y/N, returned Y/N, rep username) → Python prepare script resolves Account Id / User Id and pre-validates (reason codes) → Bulk upsert on `Legacy_Loan_Id__c` as migration admin with bypass perm set → validation SOQL V1–V6 → reconciliation sign-off. Only loans not yet returned are loaded; shipped rows as `Shipped`/`Approved`, others `Requested`/`Not_Submitted`. Date rules still enforced. Rollback before sign-off: bulk delete where `Legacy_Loan_Id__c != null`. Estimated < 500 rows (not confirmed).

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

## 8. Risks
| Risk | Impact | Mitigation |
|---|---|---|
| Rep ≠ account owner and "own accounts" undefined | Account owner may not see loan | US-119 design after PO answer |
| Reps without manager / managers without `Equipment_Loan_Manager` | Submission refused or approver cannot open the record | TK-115 queries before go-live |
| `User.ManagerId` changes after loans are created | Approvals routed to old manager | Documented; admin re-sets Sales Rep to recalc |
| Approved loan edited before shipping | Manager approved different data | Open question (D8) |
| Status lifecycle beyond `Shipped` unknown | Path, list views, report "open" definition incomplete | Open question; picklist extensible |
| Email deliverability not All Email in production | No approval/reminder emails | Post-deploy check in US-113/US-116 |
| Bypass permission left assigned after migration | Approval gate can be bypassed | US-120 AC 7 / V6: 0 assignments after load |
| Migration loaded after the daily reminder run | Loans due next day miss their reminder | Load before 07:00 or send V5 list to reps |
| Spreadsheet loans already extended beyond 30 days | Rows rejected (`DATE_RULE`) | Lumen corrects return date at source; no bypass |
| Account name matching in migration | Wrong or ambiguous account | Ask Lumen for account numbers; ambiguity → error file |

## 9. Open questions
- Status lifecycle after `Shipped` and definition of "open".
- Equipment name free text vs controlled list.
- Meaning of "own accounts" (owner vs account team) — impacts US-119.
- Spreadsheet migration in scope? If yes: volume, account identifier, rep usernames, are all "not shipped" rows unapproved, cut-over date (US-120).
- Rejection behaviour (A4), delegated approvers, re-approval after edits of an approved loan, inclusive 30 days (A3), reminder time (A5).
