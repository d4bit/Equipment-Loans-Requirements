# Solution Design – Lumen Diagnostics Demo Equipment Loans

Source: project brief (equipment-loans-requirements.pdf v1.0). Target org: SDO (production, validate-only until release). API 62.0.

## 1. Architecture overview
Fully declarative solution in the existing Sales Cloud org; no Apex, no integrations.

| Layer | Components | Stories |
|---|---|---|
| Data | Custom object `Equipment_Loan__c`, master-detail to Account | US-109 |
| UI | Tab, layout, compact layout, Lightning record page, Path, list views, Account related list | US-112 |
| Validation | Date rules, Shipped guard, Approval Status guard | US-110, US-114 |
| Automation | Before-save flow (rep/manager), approval process, scheduled reminder flow | US-111, US-113, US-116 |
| Reporting | Report "Open Equipment Loans by Equipment" | US-117 |
| Security | Permission sets, Controlled-by-Parent sharing, bypass custom permission | US-118, US-119, US-114 |
| Data | User manager clean-up, optional spreadsheet migration | TK-115, US-120 |

Loan lifecycle (confirmed part): `Requested` → (manager approval) → `Shipped` → *end status pending confirmation*.

## 2. Data model
### Equipment_Loan__c (new)
| Field | Type | Req. | Notes |
|---|---|---|---|
| `Name` | Auto Number `EL-{00000}` | — | |
| `Account__c` | Master-Detail(Account) | Yes | Relationship `Equipment_Loans`; no reparenting |
| `Equipment_Name__c` | Text(255) | Yes | May become restricted picklist (open question) |
| `Serial_Number__c` | Text(80) | Yes | Not unique |
| `Start_Date__c` | Date | Yes | |
| `Return_Date__c` | Date | Yes | |
| `Status__c` | Picklist (restricted) | Yes | `Requested` (default), `Shipped`, + pending values |
| `Approval_Status__c` | Picklist (restricted) | No | `Not_Submitted` (default), `Pending`, `Approved`, `Rejected`; system-maintained |
| `Sales_Rep__c` | Lookup(User) | No | Relationship `Equipment_Loans_As_Rep`; reminder recipient |
| `Sales_Manager__c` | Lookup(User) | No | Relationship `Equipment_Loans_As_Manager`; approver |
| `Legacy_Loan_Id__c` | Text(50), External ID, unique | No | Only if migration (US-120) confirmed |

### Standard objects used
| Object | Usage |
|---|---|
| Account | Parent of loans; sharing source |
| User | `ManagerId` → `Sales_Manager__c` |

## 3. Automation inventory
### Equipment_Loan__c
| Type | API Name | Event / Schedule | Entry condition | Story |
|---|---|---|---|---|
| Validation rule | `Return_Date_After_Start_Date` | Save | Return ≤ Start | US-110 |
| Validation rule | `Return_Date_Max_30_Days` | Save | Return − Start > 30 | US-110 |
| Validation rule | `Shipped_Requires_Approval` | Save | Status → `Shipped` (new or changed) and Approval Status ≠ `Approved`, no bypass | US-114 |
| Validation rule | `Approval_Status_System_Only` | Save | Approval Status set on create (≠ blank/`Not_Submitted`) or changed on update, no bypass | US-114 |
| Record-triggered flow (before save) | `Equipment_Loan_Before_Save_Set_Rep_Manager` | Create/Update | `ISNEW() \|\| ISCHANGED(Sales_Rep__c)` | US-111 |
| Approval process | `Equipment_Loan_Manager_Approval` | Manual submit | Status `Requested`, Approval Status blank/`Not_Submitted`/`Rejected`, Sales Manager not blank | US-113 |
| Field updates | `EL_Set_Approval_Pending/Approved/Rejected/Not_Submitted` | Approval actions | — | US-113 |
| Email alerts | `EL_Loan_Approved_Alert`, `EL_Loan_Rejected_Alert` | Final approval / rejection | Recipient `Sales_Rep__c` | US-113 |
| Schedule-triggered flow | `Equipment_Loan_Return_Reminder` | Daily 07:00 (to confirm) | Status `Shipped`, Return Date = today + 1, Sales Rep set & active | US-116 |
| Email alert | `EL_Return_Reminder_Alert` | From reminder flow | Recipient `Sales_Rep__c` | US-116 |

### Order of execution notes
- Approval field updates re-fire the before-save flow (no effect: entry condition false) and do **not** re-run custom validation rules — this is what allows `Approval_Status_System_Only` to block manual edits while the approval process still updates the field.
- No Apex triggers on `Equipment_Loan__c`; if one is ever added, document its interaction with the before-save flow.

### Email templates
| Template | Kind | Folder | Story |
|---|---|---|---|
| `Equipment_Loan_Approval_Request`, `Equipment_Loan_Approved`, `Equipment_Loan_Rejected` | Classic Text (needs `{!ApprovalRequest.*}`) | `Equipment_Loans` | US-113 |
| `Equipment_Loan_Return_Reminder` | Lightning | Equipment Loans | US-116 |

## 4. Security model
| Element | Setting | Story |
|---|---|---|
| `Equipment_Loan__c` OWD | Controlled by Parent (Account) | US-109 / US-119 |
| Account OWD | **To verify** — must not give reps access to other reps' accounts | US-119 |
| `Equipment_Loan_User` | Read/Create/Edit; FLS read-only on `Approval_Status__c`, `Sales_Rep__c`, `Sales_Manager__c` | US-118 |
| `Equipment_Loan_Manager` | Read/Create/Edit + View All; FLS read-only on `Approval_Status__c` | US-118 |
| `Equipment_Loan_Data_Migration` | Custom permission `Bypass_Equipment_Loan_Validation` only; assigned temporarily for migration | US-114 / US-120 |
| Approval submitters | All Internal Users (record visibility is the gate) | US-113 |
| Approver | Related User `Sales_Manager__c`; must hold `Equipment_Loan_Manager` (checked in TK-115) | US-113 / TK-115 |
| Edit while pending | Admin or current approver | US-113 |

## 5. Integration inventory
None (out of scope §4). Outbound email only (approval and reminder alerts); production deliverability must be All Email.

## 6. Data & volumes
- Expected volume low (demo loans of one sales team); all automation is bulk-safe for 200 records/transaction.
- TK-115: populate `User.ManagerId` for reps before go-live (38 active standard users currently lack a manager).
- US-120 (pending confirmation): upsert on `Legacy_Loan_Id__c`; Shipped rows loaded with Approval Status `Approved` by a user holding `Equipment_Loan_Data_Migration`; date rules still enforced.

## 7. Key decisions (ADR)
| # | Decision | Reason | Alternatives |
|---|---|---|---|
| D1 | `Equipment_Loan__c` master-detail to Account | "Reps see own accounts' loans" via Controlled by Parent; lifecycle owned by account | Lookup + sharing rules / Apex sharing |
| D2 | Explicit `Sales_Rep__c` / `Sales_Manager__c` lookups filled by before-save flow | Detail records have no owner; needed for approval routing and reminders | Use Account Owner (wrong if non-owner creates loan) |
| D3 | Standard approval process, single step, approver = Related User `Sales_Manager__c` | Native locking, recall, history and email; no code | Flow-based approval / Orchestrator; "Manager of submitter" (breaks when manager submits on rep's behalf) |
| D4 | Initial submitters = All Internal Users | Related User not available for submitters; record access already restricts who can see/submit | Public group to maintain; Record Creator only (blocks managers submitting for reps) |
| D5 | Two validation rules for the approval gate (Shipped guard + Approval Status system-only) | Shipped guard alone can be bypassed by anyone with FLS edit on Approval Status (admin, API) | Apex trigger; FLS only |
| D6 | Bypass via custom permission `Bypass_Equipment_Loan_Validation` in a dedicated permission set | Migration needs to load historical Shipped/Approved loans without deactivating rules | Temporarily deactivating rules in production; profile-based bypass |
| D7 | Classic email templates for approval emails | `{!ApprovalRequest.Comments}` / approval link only supported in Classic templates | Lightning templates (no rejection comments) |
| D8 | Approval not reset when an approved loan is edited (assumption) | Not requested; keeps design simple | Validation/flow resetting Approval Status on date/equipment change — pending Lumen answer |

## 8. Risks
| Risk | Impact | Mitigation |
|---|---|---|
| Account OWD wider than Private in SDO | Reps see all loans; US-119 fails | Verify in Setup; customer decision before US-119 build |
| Reps without manager / managers without `Equipment_Loan_Manager` | Submission refused or approver cannot open the record | TK-115 queries Q1–Q3 before go-live |
| `User.ManagerId` changes after loans are created | Pending/new approvals routed to old manager | Documented; admin re-sets Sales Rep to recalc |
| Approved loan edited (dates/equipment) before shipping | Manager approved different data | Open question to Lumen (D8) |
| Status lifecycle beyond `Shipped` unknown | Path, list views, report "open" definition incomplete | Open question; picklist extensible |
| Email deliverability not All Email in production | No approval/reminder emails | Post-deploy check in US-113/US-116 |
| Bypass permission left assigned after migration | Approval gate can be bypassed | Post-deploy check: no assignments outside load window |

## 9. Open questions
- Status lifecycle after `Shipped` and definition of "open".
- Equipment name free text vs controlled list.
- Meaning of "own accounts" / Account OWD.
- Spreadsheet migration in scope?
- Rejection behaviour (A4), delegated approvers, re-approval after edits of an approved loan, inclusive 30 days (A3), reminder time (A5).
