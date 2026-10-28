# US-120: Load open loans from the spreadsheet
**Status:** Cancelled (not built) · **Category:** data_migration · **Points:** 8 · **Sprint:** n/a (removed from release 1.0.0)

## Summary
Nothing changes for users. The product owner cancelled this story twice ("No migration: loading the spreadsheet is out of scope… Do not build this story"). Loans still tracked in the legacy spreadsheet will not be loaded into Salesforce at go-live.

## What was built
Nothing. No `Legacy_Loan_Id__c` field, no scripts under `scripts/data/us120/`, no permission set or layout changes, and no US-120 manifest. The branch holds only the QA test report and its evidence.

## How it works
Not applicable. AC 1–AC 7 (reconciliation, account/rep mapping, date rule, status mapping, reminders, re-runnable load, bypass removal) were never implemented. The technical specification on the work item can be reused as it is if migration comes back into scope.

## How to test
Check that the cancellation was carried out cleanly:
- **Verify:** `Equipment_Loan__c` has no `Legacy_Loan_Id__c` field (FieldDefinition query returns 0 rows in SDO).
- **Verify:** there is no `scripts/data/us120/` folder and no US-120 manifest on the branch.
- **Verify:** `Equipment_Loan_Data_Migration` has 0 permission set assignments.

QA result: all checks passed. The check-only deploy to SDO succeeded (33 components, no Apex).

## Deployment notes
None. Leave the story out of release 1.0.0, the changelog and the cut-over runbook (team decision #32). Do not assign the `Equipment_Loan_Data_Migration` permission set (US-114) to anyone for a migration.

## Known limitations
- Open loans from the spreadsheet will not be in Salesforce at go-live. Reports and US-116 return reminders cover only loans that are created in Salesforce.
- Any loans that are still out must be entered manually by reps if the business wants them tracked. This is not a requirement from the PO, only an option.
