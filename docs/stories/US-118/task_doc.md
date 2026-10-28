# US-118: Equipment Loan permission sets for reps and managers
**Status:** Ready for UAT · **Category:** security · **Points:** 3 · **Sprint:** n/a

## Summary
Access to equipment loans now comes from two permission sets. Reps can create and edit loans but can't delete them. Managers can also see and edit every loan, and they are the only business users who can change Sales Rep and Sales Manager. Approval Status is read-only for everyone.

## What was built
- `Equipment_Loan_User` (permission set): Equipment Loan Read/Create/Edit; Account Read and Contact Read; field access: `Approval_Status__c`, `Sales_Rep__c` and `Sales_Manager__c` are Read only.
- `Equipment_Loan_Manager` (permission set): same as above, plus View All on Equipment Loan, and Edit on `Sales_Rep__c` and `Sales_Manager__c`. No Delete and no Modify All.
- `Equipment_Loan_Managers` (public group): managers are added after deployment. `doesIncludeBosses = false`.
- `Equipment_Loan__c.Equipment_Loan_Managers_Edit` (owner-based sharing rule): All Internal Users → Equipment Loan Managers group, Edit.
- Manifests: `docs/stories/US-118/manifest/US-118-package.xml` and `manifest/package.xml`.

## How it works
- **AC 1:** A rep with `Equipment_Loan_User` can create a loan and edit Equipment Name and the dates. Required fields are editable through object access.
- **AC 2:** For reps, Approval Status, Sales Rep and Sales Manager are read-only, and API updates to them are rejected. There is no Delete permission.
- **AC 3:** View All lets managers read every loan. The sharing rule gives the group Edit, so managers can change dates and Sales Rep. The US-111 flow then moves Owner to the new Sales Rep.
- **AC 4:** Managers have no Delete permission, and Approval Status is read-only for them (validation rule `Approval_Status_System_Only` also applies, from US-114).
- **AC 5:** Users with neither permission set have no object access, so they don't see the tab or the related list, and SOQL fails.
- **AC 6:** Users are assigned after deployment and checked with the SOQL below.

## How to test
Test users: Rep A and Rep B (`Equipment_Loan_User`), Manager M (`Equipment_Loan_Manager` + group member), User N (no permission set).
- AC 1: Rep A creates a loan and edits it. Both saves must succeed.
- AC 2: Rep A updates Sales Rep, Sales Manager or Approval Status through the API, then tries to delete the loan. The updates must be rejected and the delete must return INSUFFICIENT_ACCESS.
- AC 3: Manager M edits Rep B's loan and changes Sales Rep to Rep A. The save must succeed and Owner must become Rep A.
- AC 4: Manager M tries to delete the loan and to edit Approval Status. Both must be rejected.
- AC 5: User N looks for the tab and the Account related list, and runs SOQL. None of them must work.
- AC 6: Run the verification SOQL below.
- Sharing rule: Setup → Sharing Settings must show "Equipment Loan Managers - Edit" (Read/Write).

## Deployment notes
- Deploy the permission sets, group and sharing rule together, in the same release as US-109. Deploying the sharing rule triggers a sharing recalculation (small volume).
- **Pre-deploy:** get Lumen's list of rep and manager usernames.
- **Post-deploy:**
  1. Assign `Equipment_Loan_User` to every rep.
  2. Assign `Equipment_Loan_Manager` to every manager (managers need only this one permission set).
  3. Add every manager to the public group Equipment Loan Managers.
- **Verification SOQL:**
  - `SELECT Assignee.Username, PermissionSet.Name FROM PermissionSetAssignment WHERE PermissionSet.Name IN ('Equipment_Loan_User','Equipment_Loan_Manager')`
  - `SELECT UserOrGroupId FROM GroupMember WHERE Group.DeveloperName = 'Equipment_Loan_Managers'`. This must return the same users as the manager assignments.
- Only formatting changed in the US-109 object and global value set files. That was done so they are included in validation. Check-only validation passed (18 components).

## Known limitations
- Tab visibility (`tabSettings` for `Equipment_Loan__c`) is not set yet. US-112 adds it once the tab exists.
- Contact Read is included because the platform requires it alongside Account Read (Person Accounts are enabled).
- No Delete for reps is an assumption that the PO still has to confirm.
- In UAT, check that a manager changing Sales Rep doesn't hit an ownership transfer error. If it does, report it to the architect.
- Out of scope: profiles, loan sharing based on account owner (US-119), report folders (US-117).
