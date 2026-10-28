# US-114: Block Shipped status until loan is approved
**Status:** Ready for UAT · **Category:** configuration · **Points:** 2 · **Sprint:** n/a

## Summary
A loan can no longer be set to Shipped (or created as Shipped) unless its manager approval is Approved. Approval Status can only be changed by the approval process, never by hand or by data loads. The only exception is a migration bypass permission.

## What was built
- **ValidationRule** `Equipment_Loan__c.Shipped_Requires_Approval` (error on `Status__c`): blocks Status `Shipped` on create or Status change when `Approval_Status__c` ≠ `Approved`.
- **ValidationRule** `Equipment_Loan__c.Approval_Status_System_Only` (error on `Approval_Status__c`): blocks a non-blank value other than `Not_Submitted` on create, and blocks any change on update.
- **CustomPermission** `Bypass_Equipment_Loan_Validation`: both rules skip it.
- **PermissionSet** `Equipment_Loan_Data_Migration`: grants only the custom permission. Not assigned by default.
- Manifests: `docs/stories/US-114/manifest/US-114-package.xml`, `manifest/package.xml`.
- Formatting-only edits to `Equipment_Loan__c.object-meta.xml` and `Equipment_Name.globalValueSet-meta.xml` so the validation deploy could run (no functional change).

## How it works
- **AC 1:** Approved loan → Status Shipped saves.
- **AC 2:** Approval Status Not_Submitted / Pending / Rejected → Shipped is blocked for every user: "This loan must be approved by your manager before it can be shipped."
- **AC 3:** Creating a loan as Shipped is blocked with the same message (UI and API).
- **AC 4:** The rule fires only on create or a Status change, so other edits to a Shipped loan still save.
- **AC 5:** Setting or changing Approval Status manually is blocked: "Approval Status is set by the approval process and cannot be changed manually." Approval process field updates don't re-run validation rules, so US-113 actions still work.
- **AC 6:** Holders of the bypass permission can load Shipped + Approved loans. The US-110 date rules still apply to them.

## How to test
No Apex tests (declarative only).
- AC 1: Approve a loan via US-113, then set Status = Shipped → saves.
- AC 2: Try Shipped with Not_Submitted, Pending (as admin) and Rejected → blocked on Status.
- AC 3: Create a loan as Shipped in the UI and with `sf data create record` → blocked.
- AC 4: Change Serial Number on a Shipped + Approved loan → saves.
- AC 5: As System Administrator, use Data Loader to insert a loan with Approved and to update Approval Status → both blocked. Then submit, approve, reject and recall → no errors.
- AC 6: Assign `Equipment_Loan_Data_Migration` and insert a Shipped + Approved loan → saves. Insert with Return Date 31 days after Start → blocked by US-110. Remove the assignment.

## Deployment notes
- Deploy with US-109 in the same release (object/GVS formatting edits depend on it).
- Post-deploy: confirm `Equipment_Loan_Data_Migration` has no assignments. Assign it only during the US-120 load window, then remove it.
- Check-only deploy to SDO passed (0 tests).

## Known limitations
- Other status transitions are not controlled (pending lifecycle confirmation).
- Not yet verified, retest in UAT: AC 5 with US-113 actions (US-113 not delivered), AC 6 with US-110 rules (US-110 not on main), and live UI checks for AC 1–4 (object not yet in SDO).
