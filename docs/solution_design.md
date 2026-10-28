# Solution Design – Lumen Diagnostics Demo Equipment Loans

Source: project brief (equipment-loans-requirements.pdf v1.0). Org: SDO (production, Enterprise Edition, validate-only until release). API 62.0.

## 1. Architecture overview
A fully declarative Sales Cloud extension:
- **Data:** new private custom object `Equipment_Loan__c` with a lookup to the customer `Account`.
- **Automation:** before-save flow for rep/manager/owner (FT-103), validation rules for date policy (FT-103), approval process routed to `Sales_Manager__c` (EP-102), scheduled flow for return reminders (EP-102).
- **UI:** custom tab, record page with Status path, Account related list (FT-103).
- **Security:** Private OWD on the loan, owner = sales rep, permission sets for reps and managers (FT-108).
- **Reporting:** custom report type and "Open Equipment Loans by Equipment" report (to be refined).
- No integrations, no Apex expected.

| Capability | Feature / Stories | Status of refinement |
|---|---|---|
| Loan object & fields | FT-103 / US-109 | Specified; waiting on PO (Status lifecycle, equipment name) |
| Date validation | FT-103 / US-110 | Ready for Sprint |
| Rep/manager auto-population | FT-103 / US-111 | Ready for Sprint |
| Loan UI | FT-103 / US-112 | Specified; waiting on PO (Status values, sales app) |
| Approval, reminder | EP-102 | Not yet refined |
| Visibility / permission sets | FT-108 / US-119 | Not yet refined |
| Report | tbd | Not yet refined |

## 2. Org facts (verified)
| Fact | Value | Impact |
|---|---|---|
| Account OWD | Public Read/Write (`DefaultAccountAccess = Edit`) | Master-detail sharing would expose all loans → lookup + Private OWD (ADR-01) |
| Person Accounts | Enabled (RTs `PersonAccount`, `SDO_PersonAccounts`) | Lookup filter restricts loans to business accounts |
| Business account record types | `Business_Account`, `SDO_Account_Simple`, `SDO_Account_Partner` | Account layouts to modify in US-112 |
| Sales apps | `LightningSales` (standard), `SDO_Sales_App` (custom) | Target app pending PO |
| Users without manager | 38 active standard users have no `ManagerId` | Manager data must be fixed before approvals go live |
| `Equipment_Loan__c` | Does not exist | New object |

## 3. Data model
### Equipment_Loan__c (new) — Private OWD, Grant Access Using Hierarchies, Name auto-number `EL-{00000}`, history tracking on
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
| `Approval_Status__c` | Picklist restricted | No | `Not_Submitted` (default), `Pending`, `Approved`, `Rejected`; system-maintained |
| `Sales_Rep__c` | Lookup(User) | No | `Equipment_Loans_As_Rep`; active standard users only |
| `Sales_Manager__c` | Lookup(User) | No | `Equipment_Loans_As_Manager`; = rep's `ManagerId`; approver |

Relationships: Account 1—n Equipment_Loan__c (lookup, restrict delete); User 1—n Equipment_Loan__c (owner, rep, manager).
Volumes: low (estimated tens per month, unconfirmed). No migration in scope (open question on spreadsheet migration).

## 4. Automation inventory
### Equipment_Loan__c
| Order | Component | Type | Event | Entry condition | Story |
|---|---|---|---|---|---|
| 1 | `Equipment_Loan_Before_Save_Set_Rep_Manager` | Record-triggered flow, before save | Create/Update | `ISNEW() \|\| ISCHANGED(Sales_Rep__c) \|\| ISCHANGED(OwnerId)` | US-111 |
| 2 | `Account_Cannot_Change` | Validation rule | Update | `AND(NOT(ISNEW()), ISCHANGED(Account__c))` | US-109 |
| 2 | `Return_Date_After_Start_Date` | Validation rule | Create/Update | `Return_Date__c <= Start_Date__c` | US-110 |
| 2 | `Return_Date_Max_30_Days` | Validation rule | Create/Update | `Return_Date__c - Start_Date__c > 30` | US-110 |
| – | Approval process (routes to `Sales_Manager__c`), Shipped-without-approval block, reminder scheduled flow | tbd | tbd | tbd | EP-102 (to refine) |

Order of execution: before-save flow runs before validation rules, so rules evaluate the flow-populated values. No Apex triggers on the object.

## 5. Security model
| Element | Design |
|---|---|
| OWD `Equipment_Loan__c` | Private / Private, hierarchies on |
| Record ownership | Owner = Sales Rep (synced by US-111) → reps see their own loans |
| Managers | View All on `Equipment_Loan__c` via `Equipment_Loan_Manager` permission set (FT-108) |
| Account owners | Access to loans on accounts they own when rep ≠ account owner — mechanism to define in US-119 (e.g. flow/Apex managed sharing); note manual shares are removed on owner change |
| Permission sets | `Equipment_Loan_User` (CRUD R/C/E, tab, FLS read on Sales Rep/Manager/Approval Status), `Equipment_Loan_Manager` (adds View All, edit Sales Rep/Manager) — FT-108 |
| FLS caveat | Field-level-required fields are not listed in permission sets |

## 6. Integration inventory
None (out of scope §4).

## 7. Key decisions (ADR)
### ADR-01 — Lookup to Account with Private OWD instead of master-detail
- **Decision:** `Account__c` is a required lookup; the loan has its own Private OWD and an owner kept equal to the Sales Rep.
- **Reason:** Account OWD is Public Read/Write; Controlled-by-Parent would grant every rep read/edit on every loan, violating §3.
- **Alternatives:** (a) master-detail + change Account OWD to Private — org-wide regression in a shared SDO; (b) master-detail + restriction rules — more complex, still edit access. 
- **Status:** Applied; PO confirmation requested (ask_human on US-109). Supersedes earlier master-detail decision.

### ADR-02 — Owner ↔ Sales Rep sync in a before-save flow
- **Decision:** One before-save flow defaults Sales Rep from Owner on create and keeps Owner and Sales Rep equal on change; computes Sales Manager from `User.ManagerId`.
- **Reason:** Private sharing needs the owner; the brief and approval/reminder logic use an explicit Sales Rep field. Before-save = no extra DML.
- **Alternatives:** Drop `Sales_Rep__c` and use Owner only (diverges from brief data model and is less readable for approvals); Apex trigger (unnecessary).

### ADR-03 — Validation rules for date policy
- **Decision:** Two validation rules (after-start, ≤ 30 days inclusive) on Return Date.
- **Reason:** Simple same-record logic; precise messages.
- **Alternatives:** One combined rule (less precise message); flow/Apex (overkill).

### ADR-04 — Declarative only
- **Decision:** No Apex in FT-103. **Reason:** brief NFR declarative-first; all needs met by flow/VR/metadata.

## 8. Risks and open points
| # | Risk / open point | Impact | Mitigation |
|---|---|---|---|
| R1 | Status lifecycle and definition of "open" unconfirmed | Path, list views, report, reminder filter | US-109/US-112 held in Ready for Refinement; additive picklist change |
| R2 | Equipment name free text vs controlled list | Report grouping quality, data model | Hold US-109 build until answered |
| R3 | 30-day rule inclusive assumed (A3) | Boundary behaviour | One-character change in `Return_Date_Max_30_Days` |
| R4 | 38 active users without manager | Approval routing fails | Manager-data task before go-live |
| R5 | Shared SDO org: modifying Account layouts / Sales app | Other demos affected | Additive changes only; validate-only until release |
| R6 | PO may reject ADR-01 | Rework of data model/sharing | Confirm before US-109 build |
| R7 | Account-owner visibility when rep ≠ account owner | Visibility gap | Design in US-119 |
| R8 | Target sales app unknown | Tab placement | ask_human on US-112 |
