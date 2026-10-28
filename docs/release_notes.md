# Release 1.1.0 (R1.1.0)

**Target org:** SDO (production, https://si1793175555706.my.salesforce.com)
**Status: VALIDATED ONLY. Nothing is deployed yet.** This was a check-only validation. The product owner deploys the release from the Release tab.

## Highlights
- Every equipment loan now needs approval from the sales rep's manager before it ships. Reps and managers get an email when a loan is approved or rejected.
- A loan's status can't be set to Shipped until the loan is approved.
- Sales reps see only the loans for their own accounts. Managers can view all loans but can't edit them.
- New report "Open Equipment Loans by Equipment", shared with all internal users.

## Delivered
| Code | Title | Category | Points |
|---|---|---|---|
| US-114 | Block Shipped status until loan is approved | Validation / data quality | n/a |
| US-119 | Loan visibility: reps see own accounts' loans only | Security / sharing | n/a |
| US-113 | Manager approval process for equipment loans | Automation / approval | n/a |
| US-117 | Open Equipment Loans by Equipment report | Reporting | n/a |

## Deployment
- **Manifest:** `manifest/package.xml` was built by `build_manifest` from `force-app/`. It covers every story's components: the approval process, workflow field updates and alerts, email templates and folder, validation rules, the custom permission, permission sets, sharing rules, the report type and the report folder and report. It also includes the base components from US-109, US-111, US-112, US-116 and US-118 that the release needs.
- **Validation:** check-only to SDO, test level RunLocalTests. ✅ Succeeded: 49 components, 0 tests run (the release has no Apex). Job `0Afak00000qDXEkCAO`. Salesforce won't quick-deploy this job because no tests ran. Deploying from the Release tab redeploys the components.
- **Pre-deployment:** none outstanding. US-113's dependencies (US-109, US-111, US-112, US-118) are in the same deployment.
- **Post-deployment (after the PO deploys):**
  1. Confirm the approval process `Equipment_Loan_Manager_Approval` is Active (US-113).
  2. Setup → Deliverability: set Access level to All Email (US-113).
  3. Check that reps who submit loans have a Manager on their User record (US-113).
  4. Confirm the `Equipment_Loan_Data_Migration` permission set has no assignments (US-114).
  5. US-119's destructive change (`destructiveChangesPost.xml`) is not needed in SDO. The `Equipment_Loan_Managers_Edit` rule was never deployed there.
  6. Verify: Equipment Loan sharing is Private/Private with no sharing rules. Equipment Loan Manager has Read and View All only. The report folder is shared with All Internal Users (View).

## Smoke test results
These checks are pre-deployment only: the validation passed and the source was reviewed. SDO is a production org, so no runtime checks were run.

| Code | Result | Note |
|---|---|---|
| US-114 | ✅ | Rule `Shipped_Requires_Approval` is active. Users with `Bypass_Equipment_Loan_Validation` can bypass it. |
| US-119 | ✅ | The sharing rules file is empty. The manager permission set has Read and View All, with no Edit, Delete or Modify All. |
| US-113 | ✅ | The approval process is active, with Sales Manager as approver. Field updates and alerts are set for submit, approve, reject and recall. |
| US-117 | ✅ | The report type, folder and report validate. The manager permission set grants Run Reports. |

## Known issues
- No runtime smoke tests have been run yet. After the PO deploys, run these checks:
  - US-113 AC 1 and AC 2 with one real rep/manager pair (delete the test loan afterwards).
  - US-114 AC 5 and AC 6 (QA couldn't run these in iteration 1).
  - Run the report as a manager on the SDO-Sales profile.
- The `Equipment_Loan_Managers` group stays in the org but nothing uses it after US-119.
- No open bugs.

## Rollback plan
- **Components to revert:** redeploy the previous versions from git (the main branch before this release) for these files:
  - the `Equipment_Loan__c` workflow file
  - the `Equipment_Loan_Manager` permission set
  - the validation rules
- **Destructive changes needed** to remove new components:
  - ApprovalProcess `Equipment_Loan__c.Equipment_Loan_Manager_Approval` (deactivate it first)
  - ValidationRules `Shipped_Requires_Approval` and `Approval_Status_System_Only`
  - CustomPermission `Bypass_Equipment_Loan_Validation`
  - PermissionSet `Equipment_Loan_Data_Migration`
  - Report `Open_Equipment_Loans_by_Equipment`, its folder and ReportType `Equipment_Loans_with_Account`
- **Data:** no data migration in this release, so there is nothing to restore. Loans that are pending approval should be recalled before the approval process is deactivated.
- **Decision:** the product owner, with advice from the release manager.
