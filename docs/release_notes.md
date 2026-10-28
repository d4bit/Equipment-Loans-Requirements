# Release 1.0.0 — Equipment Loans (first release)

**Target org:** SDO (production, https://si1793175555706.my.salesforce.com)
**Status: VALIDATED ONLY — NOT DEPLOYED.** This was a check-only validation (test level RunLocalTests). Nothing has been deployed to SDO yet. The product owner deploys it from the **Release tab** (quick deploy of validation job `0Afak00000qAuRBCA0`).

## Highlights
- New **Equipment Loan** record to track demo equipment lent to accounts (loan number, equipment, serial number, dates, status, approval status). Records are private.
- **Sales Rep** and **Sales Manager** are filled in automatically from the record owner and the owner's manager.
- Date checks: the return date must be after the start date and no more than 30 days after it.
- Equipment Loans tab, record page with a status path, "Open Loans" and "My Loans" list views. Reps and managers get access through permission sets.
- Daily 08:00 reminder email to the sales rep about loans that are due back.

## Delivered
| Code | Title | Category | Points |
|---|---|---|---|
| US-109 | Equipment Loan object and fields | Data model | — |
| US-118 | Equipment Loan permission sets for reps and managers | Security | — |
| US-111 | Auto-populate Sales Rep and Sales Manager on loan | Automation | — |
| US-110 | Loan date validation (return within 30 days of start) | Validation | — |
| US-112 | Equipment Loan user interface (tab, layout, Account related list) | UI | — |
| US-116 | Daily return reminder email to sales rep | Automation | — |
| TK-115 | Verify and populate Manager on all sales rep users | Task — **cancelled**, no components | — |
| US-120 | Load open loans from the spreadsheet | Story — **cancelled**, migration out of scope | — |

## Deployment
- **Manifest:** `manifest/package.xml` (API 62.0). It covers every story component: CustomObject, 10 CustomField, 3 ValidationRule, GlobalValueSet, 2 Flow, Workflow + WorkflowAlert, EmailFolder + EmailTemplate, CustomTab, Layout, CompactLayout, FlexiPage, PathAssistant, 2 ListView, 2 PermissionSet, Group, SharingRules/SharingOwnerRule. Nothing unrelated is included.
- **Validation:** check-only deploy to SDO ✅ Succeeded. 33 components, 0 Apex tests run (there is no Apex in this release). Job `0Afak00000qAuRBCA0`.
- **Pre-deployment steps (before the PO deploys):**
  - Get from Lumen the CSV of rep and manager **usernames** (US-118).
  - Admin populates `User.ManagerId` for all sales reps (US-111; TK-115 was cancelled, so the admin does this outside the project).
- **Post-deployment steps (after deploy):**
  1. Assign `Equipment_Loan_User` to reps and `Equipment_Loan_Manager` to managers. Add the managers to the public group **Equipment Loan Managers** (US-118).
  2. If flows were not deployed as active, activate `Equipment_Loan_Before_Save_Set_Rep_Manager` and `Equipment_Loan_Return_Reminder` (US-111, US-116).
  3. Process Automation Settings: set **Default Workflow User** and the flow error email recipient. Check that Scheduled Jobs shows the reminder's next run at 08:00 (US-116).
  4. Account page layouts used by the sales profiles (record types Business_Account, SDO_Account_Simple, SDO_Account_Partner): add the **Equipment Loans** related list with columns Loan Number, Equipment Name, Serial Number, Start Date, Return Date and Status, sorted by Return Date descending (US-112).
  5. Do **not** assign `Equipment_Loan_Data_Migration` to anyone (US-120 cancelled).

## Smoke test results
The release is check-only on a production org, so functional smoke tests (and anonymous Apex) could not run. The smoke checks are at validation level: each component validated in SDO, and SOQL confirms none of them exist there yet. Re-run the functional checks after the deploy.
| Code | Result | Note |
|---|---|---|
| US-109 | ✅ (validation) | Object, fields, value set and Account_Cannot_Change validated. After deploy: check OWD Private/Private, history tracking, 5 picklist values. |
| US-118 | ✅ (validation) | Permission sets, group and sharing rule validated. After deploy: check the assignments and group members. |
| US-111 | ✅ (validation) | Before-save flow validated. After deploy: create a loan as a rep and check Sales Rep and Sales Manager. |
| US-110 | ✅ (validation) | Both date rules validated. After deploy: a return date 31 days after start must be blocked. |
| US-112 | ✅ (validation) | Tab, layout, compact layout, record page, path, list views and tab settings validated. |
| US-116 | ✅ (validation) | Scheduled flow, alert, folder and template validated. After deploy: confirm the scheduled job. |
| TK-115 | ✅ N/A | Cancelled. Not in the manifest. |
| US-120 | ✅ N/A | Cancelled. Not in the manifest, and no migration permission set exists in SDO. |

## Known issues
- No Apex tests ran, because the release has no Apex. Behaviour is not checked by any automated test until the post-deploy smoke checks.
- The sharing rule `Equipment_Loan_Managers_Edit` ships in this release. A later story (US-119, not in this release) removes it and sets manager edit to false. Watch for the merge risks recorded for US-113, US-117 and US-119.
- The Sales Manager value depends on `User.ManagerId`. Reps without a manager get an empty Sales Manager.
- The Account related list is a manual step and is not deployed as metadata.

## Rollback plan
- This is a first release with all-new components, so nothing existing in SDO is overwritten (the Account layouts change only through the manual step).
- To roll back: deactivate both flows, remove the permission set assignments and group members, then deploy a `destructiveChanges.xml` covering the manifest components (dependents first: permission sets, FlexiPage, PathAssistant, layouts, tab, workflow, flows, sharing rules, group, email template and folder, then the object and the global value set). Remove the related list from the Account layouts by hand.
- No data migration was done, so there is no data to restore. Loan records created after go-live would be lost if the object is deleted: export them first.
- Decision owner: the product owner (Lumen Diagnostics).
