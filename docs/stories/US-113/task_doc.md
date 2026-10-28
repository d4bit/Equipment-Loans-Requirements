# US-113: Manager approval process for equipment loans
**Status:** Ready for UAT · **Category:** configuration · **Points:** 5 · **Sprint:** n/a

## Summary
Every equipment loan must now be approved by the Sales Rep's manager before it can ship. The rep clicks Submit for Approval, the Sales Manager gets an email and approves or rejects, and the rep is emailed the decision. While the request is pending, the loan is locked.

## What was built
- **ApprovalProcess** `Equipment_Loan__c.Equipment_Loan_Manager_Approval`: active, one step `Manager_Approval`, approver = `Sales_Manager__c`, delegates not allowed, recall allowed, editable by admin or the current approver while pending, record unlocked after approval or rejection.
- **WorkflowFieldUpdate**: `EL_Set_Approval_Pending`, `EL_Set_Approval_Approved`, `EL_Set_Approval_Rejected`, `EL_Set_Approval_Not_Submitted` (all set `Approval_Status__c`).
- **WorkflowAlert**: `EL_Loan_Approved_Alert`, `EL_Loan_Rejected_Alert`. Both go to `Sales_Rep__c` with sender CurrentUser.
- **EmailTemplate** (Classic Text, folder `Equipment_Loans`): `Equipment_Loan_Approval_Request`, `Equipment_Loan_Approved`, `Equipment_Loan_Rejected`.
- **EmailFolder** `Equipment_Loans` (shared with US-116).
- No Apex and no layout change: the Submit action and the Approval History list come from US-112.

## How it works
- **Entry criteria (AC 1, 6, 7):** Status = `Requested`, Approval Status blank, `Not_Submitted` or `Rejected`, and Sales Manager filled in. Any other loan gets the standard "does not meet the entry criteria…" message, and no fields change.
- **Submit (AC 1):** Approval Status = `Pending`, the record is locked and the manager gets "Equipment loan {Name} needs your approval".
- **Approve (AC 2):** Approval Status = `Approved`, Status stays `Requested`, the record is unlocked and the rep gets the "approved" email.
- **Reject (AC 3, 4):** Approval Status = `Rejected`, the record is unlocked and the rep gets the "rejected" email with the manager's comment. The rep can edit the loan and resubmit it.
- **Recall (AC 5):** Approval Status = `Not_Submitted` and the record is unlocked.
- **AC 8:** approval field updates run in system context and skip validation rules, so they work with US-114's read-only Approval Status.

## How to test
Validate: `sf project deploy validate -x docs/stories/US-113/manifest/US-113-package.xml -l NoTestRun -o SDO`. The check-only deploy together with main succeeded (43 components, 0 tests).
UAT setup: Rep A (manager = Manager M), Manager M, and Rep C with no manager.
- AC 1–5: submit, approve, reject with "Wrong unit", resubmit, recall. Check Approval Status, the lock, the emails and Approval History at each step.
- AC 6: submit an Approved loan and a Pending loan. AC 7: submit as Rep C. Expect the standard message both times.
- AC 8: repeat AC 1–5 with US-114 deployed. Expect no validation errors.
- Regression: `EL_Return_Reminder_Alert` still exists and flow `Equipment_Loan_Return_Reminder` is Active.

## Deployment notes
- Pre: US-109, US-111, US-112 and US-118 must be deployed or in the same release.
- Post: check that the approval process is Active, set Deliverability = All Email, and make sure reps have a Manager on their User record.
- Smoke test AC 1 + AC 2 with a real rep/manager pair, then have an admin delete the test loan.
- Merge risk: the shared files `workflows/Equipment_Loan__c.workflow-meta.xml` and `manifest/package.xml` must hold the merged content (US-116). Merge US-113 and US-119 in order and rebase the second one, so the removed `Equipment_Loan_Managers_Edit` sharing rule does not come back.

## Known limitations
- Runtime behaviour (emails, locks, Approval History) has not been tested yet. Production is validate-only, so these checks are for UAT.
- Open point (#28): the loan is unlocked after approval so the rep can set Shipped. The PO said "an approved loan is locked". A follow-up validation rule could block date/equipment changes once a loan is approved.
- Out of scope: multi-step approvals, auto-submission, Slack/mobile approval.
- Managers are read-only on loans (US-119). The approver acts through the approval request, not by editing the loan.
