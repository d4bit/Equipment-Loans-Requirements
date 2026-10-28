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

| Component | Type | Settings |
|---|---|---|
| `Equipment_Loan_User` | Permission set (no license) | Equipment_Loan__c Read/Create/Edit; Account Read and Contact Read; `Approval_Status__c`, `Sales_Rep__c` and `Sales_Manager__c` Read only |
| `Equipment_Loan_Manager` | Permission set (no license) | Same as User + View All; `Sales_Rep__c` and `Sales_Manager__c` Read + Edit; `Approval_Status__c` Read only |
| `Equipment_Loan_Managers` | Public group | Members are the managers; doesIncludeBosses = false |
| `Equipment_Loan__c.Equipment_Loan_Managers_Edit` | Owner-based sharing rule | All Internal Users → Equipment_Loan_Managers, Edit |

- No business user has Delete or Modify All.
- Required fields aren't listed in the permission sets; they are reachable through object access.
- Contact Read is required alongside Account Read because Person Accounts are enabled.
- Tab visibility will be added by US-112.
- The sharing rule needs OWD Private (set in US-109).
- **Post-deploy:** assign the permission sets and group membership from Lumen's lists of reps and managers. Managers get `Equipment_Loan_Manager` only.
