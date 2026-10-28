# US-119: Loan visibility: reps see own accounts' loans only
**Status:** Ready for UAT · **Category:** security · **Points:** 5 · **Sprint:** –

## Summary
Equipment loans are private to their owner. A rep sees and edits only the loans they own (Sales Rep = Owner). Managers see every loan but can't edit it. They approve loans through the approval process. Account ownership plays no part in loan visibility (per the PO answer, confirmed by the architect).

> Note: the ticket title, User Story, Background and Business Rules still describe account-owner access. The ACs and the PO answer are what count, and this document follows them.

## What was built
- `Equipment_Loan_Manager` (PermissionSet): `Equipment_Loan__c` allowEdit true → false. Read, View All and Create are kept. There is no Delete or Modify All. Field edit on `Sales_Rep__c`/`Sales_Manager__c` stays, so it only applies when a manager creates a loan.
- `Equipment_Loan__c.sharingRules-meta.xml`: rule `Equipment_Loan_Managers_Edit` removed (the file now has no rules). It is also removed from `manifest/package.xml`.
- `docs/stories/US-119/manifest/destructiveChangesPost.xml`: deletes the rule in any org that already has it.
- Unchanged: OWD Private/Private (US-109), Grant Access Using Hierarchies on. Group `Equipment_Loan_Managers` is kept but no longer used.

## How it works
- **AC 1:** The owner (Sales Rep, set by the US-111 flow) has full access except Delete.
- **AC 2:** Account owners get no access to loans they don't own. There are no manual or Apex shares.
- **AC 3:** Managers have View All, which gives them read-only access. Exception: role-hierarchy superiors of the owner inherit the owner's access.
- **AC 4:** Approving doesn't need Edit. Approval field updates run in system context.
- **AC 5:** Private/Private, hierarchies on, no Equipment Loan sharing rules.
- **AC 6:** There are no flows, triggers or rules for account owners or account transfers.
- **AC 7:** Owner-based access only, so bulk inserts are safe.

## How to test
Validate: `sf project deploy validate -x docs/stories/US-119/manifest/US-119-package.xml -o SDO` (succeeded, 32 components). Test users: Reps A and B with `Equipment_Loan_User`. Manager M with `Equipment_Loan_Manager` and a role that is not above A or B. Account X is owned by A.
- AC 1: B creates and edits a loan on X. The loan shows in B's views, search and the related list.
- AC 2: As A, the related list on X is empty and the loan URL shows "Insufficient Privileges". `Equipment_Loan__Share` has only B's Owner row.
- AC 3: M sees the loan, has no Edit or Delete button, and an API update fails.
- AC 4: B submits the loan and M approves it. Approval Status = `Approved`.
- AC 5: Sharing Settings show Private/Private, hierarchies checked and no rules.
- AC 6: Change X's owner. The loan's Owner and Sales Rep don't change.
- AC 7: B inserts 200 loans with Data Loader. There are no errors and A sees none of them.

## Deployment notes
- Pre-deploy: none.
- Post-deploy: none for SDO (the rule was never deployed there). In any org where US-118 already deployed `Equipment_Loan_Managers_Edit`, deploy `destructiveChangesPost.xml` with the release.
- Don't add managers to group `Equipment_Loan_Managers` any more.
- Merge risk: US-113's branch still has the sharing rule. Whichever of US-113 / US-119 merges second must rebase on the merged files.

## Known limitations
- Runtime checks were not run, because the object isn't in SDO yet. Run them in UAT after release 1.0.0. AC 4 depends on US-113 being deployed.
- Only admins can reassign a loan (change Sales Rep) or delete one.
- Group `Equipment_Loan_Managers` is unused (cleanup candidate).
- Out of scope: Account OWD changes, Apex sharing, account team or territory sharing, reassignment on account transfer.
