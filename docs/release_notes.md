# Release 1.0.0 — R1.0.0

**Target org:** SDO (production, https://si1793175555706.my.salesforce.com). Mode: check-only validation, test level RunLocalTests. The real deployment is a separate, human-approved step.

## Highlights
- New **Equipment Loan** record for tracking demo equipment lent to accounts: auto-numbered (EL-00000), private sharing, field history.
- Sales Rep and Sales Manager fill in automatically from the record owner and the rep's manager.
- Loan dates are enforced: Return Date must be after Start Date and no more than 30 days after it.
- Equipment Loans tab, record page with status path, list views (Open Loans, My Loans), and permission sets for reps and managers.
- A daily email reminds the Sales Rep the day before a shipped loan is due back, and logs the email as an activity on the loan.

## Delivered
| Code | Title | Category | Points |
|---|---|---|---|
| US-109 | Equipment Loan object and fields | configuration | 5 |
| US-118 | Equipment Loan permission sets for reps and managers | security | 3 |
| US-111 | Auto-populate Sales Rep and Sales Manager on loan | configuration | 3 |
| US-110 | Loan date validation (return within 30 days of start) | configuration | 1 |
| US-112 | Equipment Loan user interface (tab, layout, Account related list) | configuration | 3 |
| TK-115 | Verify and populate Manager on all sales rep users | script | 2 (cancelled by PO, nothing delivered) |
| US-116 | Daily return reminder email to sales rep | configuration | 5 |

Not included: US-120 (spreadsheet migration). The PO cancelled it. US-113 (approval) is not in this release.

## Deployment
- Manifest `manifest/package.xml` built from force-app (API 62.0) with 33 components: CustomObject, 10 CustomFields, 3 ValidationRules, GlobalValueSet, 2 Flows, Workflow + WorkflowAlert, EmailTemplate + folder Equipment_Loans, CustomTab, Layout, CompactLayout, FlexiPage, PathAssistant, 2 ListViews, 2 PermissionSets, Group, SharingRules/SharingOwnerRule.
- The manifest generator previously dropped some components (team note #34). That gap is fixed: every story component is present, and the manifest contains nothing unrelated.
- **Validation:** check-only deploy to SDO ✅ succeeded. 33 components, 0 tests run (the release has no Apex).
- **Pre-deployment:** deploy all stories together in one manifest. US-111, US-112 and US-116 depend on US-109. Get the CSV of rep and manager usernames from Lumen.
- **Post-deployment steps (manual):**
  1. Assign the `Equipment_Loan_User` permission set to reps and `Equipment_Loan_Manager` to managers. Add the managers to the public group `Equipment_Loan_Managers`.
  2. Account page layouts used by the sales profiles (Business_Account, SDO_Account_Simple, SDO_Account_Partner): add the **Equipment Loans** related list with the columns given in US-112.
  3. Process Automation Settings: set the Default Workflow User, and set the flow error email recipient to the Lumen admin.
  4. Check that both flows are Active. Check that Setup → Scheduled Jobs shows the next run of `Equipment_Loan_Return_Reminder` at 08:00 Pacific.
  5. Admin keeps `User.ManagerId` filled in for every sales rep (38 active users currently have no manager).

## Smoke test results
SDO is a production org and this run is validate-only, so smoke checks are at validation and metadata level. Run the runtime checks (create a loan, date rules, reminder) after the real deployment.

| Code | Result | Note |
|---|---|---|
| US-109 | ✅ | Object, fields, value set and validation rule accepted by validation |
| US-118 | ✅ | Permission sets, group and sharing rule now in the manifest and validated |
| US-111 | ✅ | Before-save flow is Active, triggers on create/update and sets Rep, Owner and Manager |
| US-110 | ✅ | Both date validation rules are Active; exactly 30 days is allowed |
| US-112 | ✅ | Tab, layout, compact layout, record page, path and list views validated |
| TK-115 | ✅ | Cancelled; nothing to deploy |
| US-116 | ✅ | Scheduled flow, email alert and template validated |

## Known issues
- In metadata, the reminder flow's start time is stored as `08:00:00.000Z`. After deployment, confirm in Scheduled Jobs that it runs at 08:00 Pacific, and adjust the flow start if it does not.
- Loans whose Sales Rep has no manager get a blank Sales Manager until the admin fills in `User.ManagerId`.
- Adding the Account related list is a manual step and is not covered by the metadata.

## Rollback plan
- All components are new, so rollback means removing them. Use a destructiveChanges.xml with the same members as package.xml. Order: deactivate the flows first, then remove permission sets, sharing rules and group, then UI components (FlexiPage, PathAssistant, Layout, Tab), then workflow alert and email template/folder, and finally the object and global value set.
- No data migration is in this release. If loans were already created, export Equipment_Loan__c records before deleting the object.
- Revert the Account layout related lists by hand.
- Decision owner: Lumen product owner, with the release manager.
