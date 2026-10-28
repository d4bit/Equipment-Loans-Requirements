# US-110: Loan date validation (return within 30 days of start)
**Status:** Ready for UAT · **Category:** configuration · **Points:** 1 · **Sprint:** n/a

## Summary
A loan can only be saved if its Return Date is after its Start Date and no more than 30 days later. Exactly 30 days is allowed (PO confirmed). The rule applies to every user, including admins, and to API loads.

## What was built
- `Equipment_Loan__c.Return_Date_After_Start_Date`: validation rule that blocks Return Date ≤ Start Date.
- `Equipment_Loan__c.Return_Date_Max_30_Days`: validation rule that blocks Return Date − Start Date > 30.
- Formatting-only touches to `Equipment_Loan__c` object metadata and the `Equipment_Name` GVS. These have no functional effect and were only needed to validate the story while US-109 is not yet in SDO.
- Manifests: `docs/stories/US-110/manifest/US-110-package.xml` and `manifest/package.xml`.

## How it works
| AC | Behaviour |
|---|---|
| 1 | 2025-03-01 → 2025-03-15 saves |
| 2 | 2025-03-01 → 2025-03-31 (30 days) saves. The rule is inclusive (`> 30`) |
| 3 | 2025-03-01 → 2025-04-01 is blocked with "Return Date must be no more than 30 days after Start Date." on Return Date |
| 4 | Same day or earlier is blocked with "Return Date must be after Start Date." The two rules never fire together |
| 5 | The rules run on every edit, whichever date changes |
| 6 | No bypass. A violating API record fails with FIELD_CUSTOM_VALIDATION_EXCEPTION and valid records in the same batch still save |

Both formulas have ISBLANK guards. Blank dates are already caught by required fields (US-109).

## How to test
No Apex. Check-only deploy succeeded (0 tests). In UAT after US-109 is deployed:
- AC 1–4: create loans with the dates above and verify the save result and message under Return Date.
- AC 5: edit a valid loan so that the return date is 31+ days after the start date. Verify the save is blocked.
- AC 6: as System Administrator, insert one valid loan and one 31-day loan via Data Loader. Verify one success and one validation error.

## Deployment notes
- No pre- or post-deployment steps and no permission sets.
- Deploy together with or after US-109 (the story manifest only validates on its own once US-109 is deployed).
- Verify in Object Manager → Equipment Loan → Validation Rules that both rules are Active.

## Known limitations
- Not tested live: SDO is validate-only. Run the manual tests in UAT.
- No check on past start dates, and no extensions beyond 30 days.
- No bypass permission. If data migration is approved later, add `Bypass_Equipment_Loan_Validation`.
