# Project Brief – Lumen Diagnostics Demo Equipment Loans

Source: *equipment-loans-requirements.pdf* v1.0

## 1. Goal
Replace the sales team's spreadsheet with an Equipment Loan process in the existing Salesforce Sales Cloud org, so that loans are approved before shipping, equipment comes back on time and managers can see what is out.

## 2. Scope
| # | Capability | Source |
|---|---|---|
| 1 | Equipment Loan record linked to the customer Account (equipment name, serial number, start date, return date) | §2.1 |
| 2 | Validation: return date after start date and at most 30 days after it | §2.1 |
| 3 | Manager approval before status moves Requested → Shipped | §2.2 |
| 4 | Email reminder to the sales rep one day before return date while the loan is still Shipped | §3 |
| 5 | Report of open loans grouped by equipment name with account and return date | §3 |
| 6 | Visibility: reps see loans of their own accounts only; managers see all loans | §3 |

## 3. Personas
| Persona | Description | Main needs |
|---|---|---|
| Sales Rep | Owns hospital accounts, requests demo equipment loans | Create loans quickly, submit for approval, get reminded of returns |
| Sales Manager | Manager (User.ManagerId) of sales reps | Approve/reject loans, see all loans, report on open loans |
| Salesforce Administrator | Lumen internal admin | Maintain users/managers, permission sets, reports |

## 4. Business processes
### 4.1 As-is
1. Rep agrees a demo with a hospital.
2. Rep records the loan in a shared spreadsheet.
3. Equipment is shipped; nobody is reminded of the return date; managers have no reliable overview.

### 4.2 To-be – Loan request & approval
1. Rep opens the hospital Account and creates an Equipment Loan (equipment name, serial number, start date, return date). Status defaults to `Requested`.
2. System validates dates (return > start, return ≤ start + 30 days).
3. Rep submits the loan for approval; it is routed to the rep's manager (User.ManagerId) and locked.
4. Manager approves or rejects. Approved → loan can be set to `Shipped`. Rejected → stays `Requested` (assumption; see open questions).
5. Any attempt to set `Shipped` without approval is blocked.

### 4.3 To-be – Follow-up
1. Daily, the system finds loans with Status `Shipped` and return date = tomorrow.
2. An email reminder is sent to the loan's sales rep.
3. Rep arranges the return and updates the status (end-of-loan status to be confirmed).

### 4.4 To-be – Visibility
1. Reps see only loans on accounts they own.
2. Managers see every loan and run the "Open Equipment Loans by Equipment" report.

## 5. Data entities
| Entity | Type | Notes |
|---|---|---|
| Account | Standard (exists) | Customer hospital; parent of loans |
| User | Standard (exists) | `ManagerId` used for approval routing |
| Equipment_Loan__c | **New** custom object (confirmed not present in org) | Master-detail to Account; fields: Equipment_Name__c, Serial_Number__c, Start_Date__c, Return_Date__c, Status__c, Approval_Status__c, Sales_Rep__c, Sales_Manager__c |

## 6. Integrations
None. Integrations with external systems are explicitly out of scope (§4).

## 7. Non-functional requirements
- Declarative-first (Flows, validation rules, approval process); no Apex expected. If Apex is introduced: API 62.0, ≥75% coverage.
- Access through permission sets, not profiles.
- Automation bulk-safe (200 records per transaction).
- Org is a production SDO org: validate-only until release.

## 8. Assumptions
- A1: The sales rep of a loan is the user who creates it (stored in `Sales_Rep__c`, editable by managers/admin).
- A2: Equipment name and serial number are entered as text (no product/asset catalogue) — pending confirmation.
- A3: "30 days" is inclusive (return date may be exactly start date + 30).
- A4: Rejected loans stay `Requested` and may be edited and resubmitted.
- A5: Reminder is sent once, on the calendar day before the return date (weekends included), in the morning org time.
- A6: The rep (not the manager) updates the status to `Shipped` after approval.
- A7: "Own accounts" = accounts where the rep is Account Owner.

## 9. Out of scope
- Shipping logistics, invoicing, integrations with external systems (§4).
- Equipment inventory/availability management and double-booking checks (not requested).
- Overdue escalation after the return date (not requested).

## 10. Open questions (recorded with ask_human)
- Full Status lifecycle (values after Shipped; Cancelled/Rejected?) and definition of "open".
- Equipment name: free text vs. controlled list (report grouping quality).
- Account org-wide default / meaning of "own accounts" (owner vs. account team).
- Migration of existing spreadsheet loans.
- Rejection behaviour, missing manager, inclusive 30-day rule, reminder timing.
