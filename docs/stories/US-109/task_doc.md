# US-109: Equipment Loan object and fields
**Status:** Ready for UAT · **Category:** configuration · **Points:** 5 · **Sprint:** n/a

## Summary
Adds the Equipment Loan record, always linked to a business account, that replaces the sales spreadsheet. Each loan holds the equipment, serial number, loan dates, status and approval status. Loans are private to the owning sales rep.

## What was built
- `Equipment_Loan__c` custom object: label Equipment Loan(s), Name "Loan Number" auto-number `EL-{00000}`, OWD Private/Private, Reports/Activities/History/Search/Sharing/Bulk/Streaming enabled.
- Global value set `Equipment_Name`: Ultrasound U1, Patient Monitor M5, Blood Analyzer A200, Infusion Pump P3, ECG Device E12.
- Fields: `Account__c` (required lookup, Restrict delete, business accounts only), `Equipment_Name__c` (restricted picklist), `Serial_Number__c` (Text 80, not unique), `Start_Date__c`, `Return_Date__c`, `Status__c` (Requested default, Approved, Shipped, Returned, Rejected, Cancelled), `Approval_Status__c` (Not_Submitted default, Pending, Approved, Rejected), `Sales_Rep__c` (active internal user), `Sales_Manager__c`.
- History tracking on 9 custom fields + Owner.
- Validation rule `Account_Cannot_Change`.
- Manifests: `docs/stories/US-109/manifest/US-109-package.xml`, `manifest/package.xml`.

## How it works
- AC 1–2: object exists with the features above; sharing Private, hierarchy access granted.
- AC 3: Account is required; Person Accounts are rejected with "Equipment loans can only be linked to business accounts."
- AC 4: Equipment Name, Serial Number, Start Date and Return Date are required at field level (UI and API).
- AC 5: new loans default to Status `Requested`, Approval Status `Not_Submitted`.
- AC 6: restricted picklists reject unknown values.
- AC 7: changing Account after create fails with "The Account of an Equipment Loan cannot be changed."
- AC 8: an account with loans cannot be deleted (Restrict).
- AC 9: serial numbers can repeat across loans.
- Design: lookup instead of master-detail because Account OWD is Public Read/Write (PO confirmed).

## How to test
Validate: `sf project deploy validate -x docs/stories/US-109/manifest/US-109-package.xml -l RunLocalTests -o SDO`. Then one check per AC as listed in `docs/stories/US-109/how-to-test.md` (Object Manager settings, Sharing Settings, required/lookup-filter errors, defaults, restricted-picklist errors, account change blocked, account delete blocked, duplicate serial allowed).

## Deployment notes
- No pre/post-deployment steps. No Apex.
- No permission set in this story; access comes with FT-108 (`Equipment_Loan_User`, `Equipment_Loan_Manager`).

## Known limitations
- No layouts, tab, app or list views (US-112); no date validation (US-110); no rep/manager auto-fill (US-111); no sharing (FT-108/US-119).
- Auto-number starting value is the platform default.
- Person Accounts cannot hold loans (assumption: hospitals are business accounts).
