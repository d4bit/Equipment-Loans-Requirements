# US-111: Auto-populate Sales Rep and Sales Manager on loan
**Status:** Ready for UAT · **Category:** configuration · **Points:** 3 · **Sprint:** n/a

## Summary
Sales Rep and Sales Manager on an Equipment Loan now fill in automatically. The loan owner and the Sales Rep always stay the same person, and Sales Manager is taken from the rep's manager on the user record.

## What was built
- Flow `Equipment_Loan_Before_Save_Set_Rep_Manager`: record-triggered, before save (Fast Field Updates), runs on create and update of `Equipment_Loan__c`, Active, API 62.0.
- Description-only edits to `Equipment_Loan__c`, `Sales_Rep__c`, `Sales_Manager__c` and GVS `Equipment_Name`. These edits let the check-only validation run while US-109 is not in SDO.
- Manifests: `docs/stories/US-111/manifest/US-111-package.xml`, `US-111-with-US-109-validate-package.xml`, and the updated `manifest/package.xml`.

## How it works
Entry condition: `OR(ISNEW(), ISCHANGED(Sales_Rep__c), ISCHANGED(OwnerId))`.
- **AC 1 – rep defaulted:** if Sales Rep is blank, it is set to the owner. The owner is the creator unless an owner is given. Sales Manager is set to the rep's `ManagerId`.
- **AC 2 / AC 3 – explicit rep or rep change:** the owner is set from the rep and the manager is recalculated.
- **AC 4 – Change Owner:** the new owner becomes the Sales Rep and the manager is recalculated.
- **AC 5 – no manager:** Sales Manager stays blank and the loan saves.
- **AC 6 – unrelated edits:** the flow does not run, so the manager is not recalculated.
- **AC 7 – bulk:** the flow runtime bulkifies the Get Records on User (1 SOQL per batch, no DML).
- If both rep and owner change in one save, the rep wins. If a manager clears Sales Rep on edit, it is reset from the current owner.
- If the User lookup fails, the fault path shows a custom error: "The Sales Manager could not be determined from the Sales Rep…"
- Changes from the spec: the two outcomes "rep blank" and "owner changed" are merged into one outcome, `Rep_Blank_Or_Owner_Changed`, because Auto-Layout requires it.

## How to test
Validate with `sf project deploy validate -x docs/stories/US-111/manifest/US-111-with-US-109-validate-package.xml -l RunLocalTests -o SDO`.

Run the functional tests in a sandbox with US-109 and US-111 deployed. Use Rep A (manager M), Rep B (manager N) and Rep C (no manager):
- **AC 1:** As Rep A, create a loan with Sales Rep blank. **Verify:** Sales Rep = Owner = Rep A, Sales Manager = M.
- **AC 2:** As admin, create a loan with Sales Rep = Rep B. **Verify:** Owner = Rep B, Sales Manager = N.
- **AC 3:** Change Sales Rep from Rep A to Rep B. **Verify:** Owner = Rep B, Sales Manager = N.
- **AC 4:** Use Change Owner to move a loan from Rep A to Rep B. **Verify:** Sales Rep = Rep B, Sales Manager = N.
- **AC 5:** As Rep C, create a loan. **Verify:** it saves and Sales Manager is blank.
- **AC 6:** Change Rep A's manager to N, then edit only Serial Number. **Verify:** Sales Manager is still M.
- **AC 7:** Load 200 loans with Sales Rep blank. **Verify:** all succeed, Sales Rep = Owner, and Sales Manager is correct.

## Deployment notes
- **Pre:** deploy US-109 first or in the same manifest. If the descriptions conflict on merge, take the US-111 text. Fill in the Manager field on the user record for every sales rep (38 active users currently have none).
- **Post:** if flows are not deployed as active, activate the flow in Setup → Flows.
- FLS comes from US-118: Read for `Equipment_Loan_User`, Read/Edit for `Equipment_Loan_Manager`.
- No Apex, so no tests run. The review tool may wrongly report "Apex tests failed" (team note #20).

## Known limitations
- If a user's manager changes later, existing loans are not updated.
- Delegated approvers are not supported.
- If a rep has no manager, the loan has no Sales Manager and approval submission will fail.
