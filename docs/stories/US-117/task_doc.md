# US-117: Open Equipment Loans by Equipment report
**Status:** Ready for UAT · **Category:** Reporting · **Points:** 2 · **Sprint:** n/a

## Summary
Sales Managers now have a report, "Open Equipment Loans by Equipment", in the "Equipment Loans" report folder. It lists every open loan (Requested, Approved, Shipped) grouped by equipment, with the account and return date, earliest return first. Only managers can run it.

## What was built
- **ReportType** `Equipment_Loans_with_Account`: primary object `Equipment_Loan__c`, with Account fields added through the `Account__c` lookup. Category `accounts`.
- **Report folder** `Equipment_Loan_Reports` (label "Equipment Loans"): shared View with All Internal Users through `folderShares`.
- **Report** `Equipment_Loan_Reports/Open_Equipment_Loans_by_Equipment`: summary format, scope organization.
- **PermissionSet** `Equipment_Loan_Manager` (modified): adds system permission `RunReports`. The description is shortened to 255 characters or fewer.
- **Manifests:** `manifest/package.xml` and `docs/stories/US-117/manifest/US-117-package.xml`.

## How it works
- **AC 1:** Managers get Run Reports from `Equipment_Loan_Manager`, so the report runs even on the SDO-Sales profile.
- **AC 2:** The report is grouped by `Equipment_Name__c` (A→Z) and shows a record count per group plus a grand total. Columns, in order: Loan Number, Account Name, Return Date, Start Date, Serial Number, Status, Approval Status, Sales Rep. Rows are sorted by Return Date ascending.
- **AC 3:** The filter is `Status__c` equals Requested, Approved, Shipped. The date filter covers all time.
- **AC 4:** Record visibility follows sharing. Managers have View All (US-118), so they see every loan.
- **AC 5:** `Equipment_Loan_User` and the rep profiles are unchanged, so reps cannot run reports.
- **AC 6:** Users without object access cannot use the report type, so they see no loan data.

## How to test
- **Validate:** `sf project deploy validate -x docs/stories/US-117/manifest/US-117-package.xml -l NoTestRun -o SDO`. This passed: 50 components, no Apex.
- **Test data:** create loans for two reps with each status: Requested, Approved, Shipped, Returned, Rejected and Cancelled.
- **AC 1–2:** As a manager, open Reports → "Equipment Loans" → run the report. Check the grouping, columns and sort.
- **AC 3:** Check that only Requested, Approved and Shipped loans appear.
- **AC 4:** Compare the report total with this query run as admin: `SELECT COUNT() FROM Equipment_Loan__c WHERE Status__c IN ('Requested','Approved','Shipped')`.
- **AC 5:** Check that neither `Equipment_Loan_User` nor SDO-Sales has Run Reports.
- **AC 6:** As a user without the loan permission sets, check that the report shows no data.

## Deployment notes
- **Deployment order:** US-109 → US-118 → US-117.
- **Manual steps:** none, before or after deployment.
- **Permission set:** assign `Equipment_Loan_Manager` to managers. This is already the case if US-118 was set up.
- **Merge risk with US-119:** in `Equipment_Loan_Manager`, keep US-119's allowEdit=false and add US-117's RunReports. Refresh the description. Remove SharingOwnerRule `Equipment_Loan_Managers_Edit` from `manifest/package.xml`.

## Known limitations
- Not tested yet (left for UAT): the report runs for AC 1, AC 4 and AC 6. SDO is validate-only.
- If a new "open" status is added, it must also be added to the report filter.
- Out of scope: dashboards, charts, scheduled subscriptions, overdue highlighting, history reports.
