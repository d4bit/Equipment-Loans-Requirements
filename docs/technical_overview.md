# Technical overview

## Equipment Loan (Equipment_Loan__c)

Introduced by US-109. 100% declarative.

**Object:** Name = Loan Number, auto-number `EL-{00000}`. OWD Private (internal/external), hierarchy access on. Reports, Activities, Field History, Search, Sharing, Bulk API, Streaming API enabled. No record types. Lookup (not master-detail) to Account because Account OWD is Public Read/Write.

| Field | Type | Required | Notes |
|---|---|---|---|
| `Account__c` | Lookup(Account) | Yes | Delete = Restrict; filter `IsPersonAccount = False`; relationship `Equipment_Loans` |
| `Equipment_Name__c` | Restricted picklist (GVS `Equipment_Name`) | Yes | API = label, contains spaces |
| `Serial_Number__c` | Text(80) | Yes | Not unique |
| `Start_Date__c` / `Return_Date__c` | Date | Yes | |
| `Status__c` | Restricted picklist | Yes | Requested (default), Approved, Shipped, Returned, Rejected, Cancelled; open = first three |
| `Approval_Status__c` | Restricted picklist | No | Not_Submitted (default), Pending, Approved, Rejected; approval-process only |
| `Sales_Rep__c` | Lookup(User) | No | Filter active Standard users |
| `Sales_Manager__c` | Lookup(User) | No | Approver (EP-102) |

History tracked: 9 custom fields + Owner (10/20).

**Validation rules:** `Account_Cannot_Change` — `AND(NOT(ISNEW()), ISCHANGED(Account__c))`.

**Global value set:** `Equipment_Name` — Ultrasound U1, Patient Monitor M5, Blood Analyzer A200, Infusion Pump P3, ECG Device E12.

**Permissions:** none yet (FT-108). Field-level required fields must not be in permission set field permissions.

**Deployment:** no manual steps; manifest `docs/stories/US-109/manifest/US-109-package.xml`.
