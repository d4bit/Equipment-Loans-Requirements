# US-112: Equipment Loan user interface (tab, layout, Account related list)
**Status:** Ready for UAT · **Category:** configuration · **Points:** 3 · **Sprint:** n/a

## Summary
Sales reps and managers can open an "Equipment Loans" tab from the App Launcher. They can also create loans from a hospital account, with the Account filled in for them. Each loan has a record page with a Status path, a compact highlights panel and read-only approval fields.

## What was built
- **CustomTab** `Equipment_Loan__c` (Box icon). It is not added to any app (PO decision). It is visible through `tabSettings` in `Equipment_Loan_User` and `Equipment_Loan_Manager`.
- **Layout** `Equipment_Loan__c-Equipment Loan Layout`:
  - Sections: "Loan Details", "Approval" (`Approval_Status__c` read-only) and "System Information".
  - Buttons removed: `ChangeOwnerOne` and `ChangeRecordType`.
  - Actions: Edit, Submit, Delete.
  - Related lists: `RelatedProcessHistoryList`, `RelatedHistoryList`, `RelatedEntityHistoryList`.
- **CompactLayout** `Equipment_Loan_Compact` (Name, Account__c, Equipment_Name__c, Status__c, Return_Date__c), assigned on the object.
- **FlexiPage** `Equipment_Loan_Record_Page`: highlights panel, Path, details and related lists. It is the desktop default through the View `actionOverrides`.
- **PathAssistant** `Equipment_Loan_Status_Path` on `Status__c`, active.
- **ListViews** `Open_Loans` (scope Everything; Status = Requested, Approved or Shipped) and `My_Loans` (scope Mine).
- **Manual (post-deploy):** "Equipment Loans" related list on the business Account layouts.

## How it works
- **AC 1:** Users with a loan permission set find "Equipment Loans" in the App Launcher. "Open Loans" lists every visible loan in Requested, Approved or Shipped. "My Loans" lists loans the user owns (owner = Sales Rep, per US-111).
- **AC 2:** The Account related list shows Loan Number, Equipment Name, Serial Number, Start Date, Return Date and Status, sorted by Return Date descending. Clicking New prefills the Account.
- **AC 3:** The record page shows the Loan Details and Approval sections, the Status path, and the Approval History, Activity History and Equipment Loan History related lists.
- **AC 4:**
  - Approval Status is read-only on the layout for everyone.
  - Sales Rep and Sales Manager are read-only for reps through field-level security in `Equipment_Loan_User`.
  - Change Owner is not available.
- **AC 5:** The highlights panel shows Loan Number, Account, Equipment Name, Status and Return Date.

## How to test
No unit tests (declarative only). Validate with: `sf project deploy validate -x docs/stories/US-112/manifest/US-112-package.xml -l NoTestRun -o SDO`.
- **AC 1:** Log in as a rep → App Launcher → Equipment Loans. **Verify:** both list views filter correctly.
- **AC 2:** Do this after the manual step. Open a business Account → Equipment Loans → New. **Verify:** the columns are correct and the Account is prefilled.
- **AC 3:** Open a loan. **Verify:** the sections, Path and related lists are there.
- **AC 4:** As a rep, click Edit. **Verify:** Approval Status, Sales Rep and Sales Manager can't be edited, and there is no Change Owner action.
- **AC 5:** **Verify:** the highlights panel fields are correct.

## Deployment notes
- **Pre-deploy:** none. Release US-112 together with US-109, US-111 and US-118.
- **Post-deploy:**
  1. Add the **Equipment Loans** related list to the Account layouts for record types `Business_Account`, `SDO_Account_Simple` and `SDO_Account_Partner` that the sales profiles use. Columns: Loan Number, Equipment Name, Serial Number, Start Date, Return Date, Status. Sort by Return Date descending. Keep the New button.
  2. If the Account page uses Dynamic Related Lists, add one for Equipment Loans in Lightning App Builder instead.
  3. Confirm `Equipment_Loan_Record_Page` is the org default.
  4. Confirm Path Settings is enabled.
- **Permission sets:** assign `Equipment_Loan_User` (reps) or `Equipment_Loan_Manager` (managers).

## Known limitations
- The Account related list is a manual step. The SDO Account layouts can't be retrieved, and deploying them would overwrite shared layouts. QA could not test AC 2; verify it in UAT.
- No live screenshots were taken because the org is validate-only. Repeat the UI checks in UAT.
- Rejected and Cancelled appear as steps on the linear path.
- No custom LWC and no mobile-specific layouts.
