# US-114: Block Shipped status until loan is approved
**Status:** Ready for UAT · **Category:** configuration · **Points:** 2 · **Sprint:** n/a

## Summary
A loan can no longer be marked Shipped until its manager has approved it, and nobody can set Approval Status by hand. The rules apply to every user, including managers, admins and API loads. The only exception is a dedicated migration bypass permission.

## What was built
- **ValidationRule** `Equipment_Loan__c.Shipped_Requires_Approval`: blocks Status = `Shipped` on create, or when Status changes to Shipped, unless Approval Status = `Approved`. The error shows on Status.
- **ValidationRule** `Equipment_Loan__c.Approval_Status_System_Only`: blocks creating a loan with an Approval Status other than blank or `Not_Submitted`, and blocks any manual change to Approval Status. The error shows on Approval Status.
- **CustomPermission** `Bypass_Equipment_Loan_Validation`: skips both rules above.
- **PermissionSet** `Equipment_Loan_Data_Migration`: grants only the custom permission. It is not assigned by default.
- Manifests: `docs/stories/US-114/manifest/US-114-package.xml` and `manifest/package.xml`.
- Validation-only touches with no functional change: `Equipment_Loan__c.object-meta.xml` and `Equipment_Name.globalValueSet-meta.xml`.

## How it works
- **AC 1:** An approved loan can be set to Shipped.
- **AC 2:** A loan with Approval Status Not Submitted, Pending or Rejected can't be set to Shipped. The user sees: "This loan must be approved by your manager before it can be shipped."
- **AC 3:** A loan can't be created directly as Shipped (UI or API). The user gets the same message.
- **AC 4:** Other edits to Shipped + Approved loans still save, because the rule fires only when Status is new or changed.
- **AC 5:** Setting or changing Approval Status manually is blocked with "Approval Status is set by the approval process and cannot be changed manually." Approval process field updates don't re-run validation rules, so the US-113 actions still work.
- **AC 6:** A user with the bypass permission can load a loan as Shipped + Approved. The US-110 30-day date rules still apply to that user.

## How to test
- AC 1: Set an approved loan to Shipped. It saves.
- AC 2: Try Shipped on Not Submitted, Pending and Rejected loans. All are blocked with the Status message.
- AC 3: Create a loan as Shipped in the UI and via `sf data create record`. Both are blocked.
- AC 4: Change the Serial Number on a Shipped + Approved loan. It saves.
- AC 5: As an admin, use Data Loader to insert a loan with Approval Status = Approved, then update an existing loan's Approval Status. Both are blocked. Then run submit, approve, reject and recall. None of them error.
- AC 6: Assign `Equipment_Loan_Data_Migration` to a test user. Insert a loan as Shipped + Approved: it saves. Insert one with a return date more than 30 days after the start date: US-110 blocks it. Remove the assignment afterwards.

## Deployment notes
- No Apex. The check-only validation to SDO succeeded (37 components).
- Deploy together with US-109 in the same release.
- Post-deploy: confirm `Equipment_Loan_Data_Migration` has no assignments. US-120 is cancelled, so the permission set stays unassigned.
- UAT: verify AC 5 (with the US-113 approval actions) and AC 6 (with the US-110 date rules). QA could not test these in SDO.

## Known limitations
- Other status transitions are not controlled; they are pending lifecycle confirmation.
- There are no live UI screenshots, because the object is not yet deployed to SDO.
