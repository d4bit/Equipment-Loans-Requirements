# Delivery Plan – Lumen Diagnostics Demo Equipment Loans (ELR)

## 1. Summary
- **Scope:** 11 user stories + 1 task in 6 features (FT-103 to FT-108). Declarative only (ADR-04), Salesforce API 62.0. Target org SDO is production: validate-only until release.
- **Total:** 44 story points over **3 sprints** of 2 weeks (capacity 20 pts per sprint): 19 + 17 + 8.
- **Order:** data model and security first (US-109, US-118), then automation and UI, then sharing, approval, reminders and reporting. Migration (US-120) goes last because it needs everything else deployed, US-119 most of all (note #13).
- **Key risks:** open PO answers that block the start of US-109 (equipment name free text vs controlled list R2, status lifecycle R1, ADR-01 acceptance R6). Migration scope is not confirmed (FT-105 is in "Requires additional information"). No org-wide email address yet (R9). 38 users have no manager (R4).

## 2. Estimation and prioritisation
WSJF = (Business value + Time criticality + Risk reduction) / story points.

| Rank | Code | Title | Pts | MoSCoW | BV | TC | RR | WSJF | Prio | Depends on | Sprint |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | US-109 | Equipment Loan object and fields | 5 | Must | 8 | 8 | 10 | 5.2 | 1 | – | 1 |
| 2 | US-118 | Permission sets for reps and managers | 3 | Must | 7 | 8 | 9 | 8.0 | 1 | US-109 | 1 |
| 3 | US-111 | Auto-populate Sales Rep / Sales Manager | 3 | Must | 7 | 6 | 6 | 6.33 | 1 | US-109 | 1 |
| 4 | US-110 | Loan date validation (30 days) | 1 | Must | 6 | 5 | 3 | 14.0 | 1 | US-109 | 1 |
| 5 | US-114 | Block Shipped until approved | 2 | Must | 7 | 6 | 5 | 9.0 | 1 | US-109 | 1 |
| 6 | US-112 | UI: tab, layout, Account related list | 3 | Must | 7 | 6 | 3 | 5.33 | 2 | US-109, US-111 | 1 |
| 7 | TK-115 | Verify/populate Manager on sales reps | 2 | Must | 5 | 7 | 8 | 10.0 | 1 | US-118 | 1 |
| 8 | US-119 | Loan visibility (account-owner shares) | 5 | Must | 8 | 6 | 7 | 4.2 | 1 | US-109, US-111, US-118 | 2 |
| 9 | US-113 | Manager approval process | 5 | Must | 9 | 7 | 6 | 4.4 | 1 | US-109, US-111, US-114, US-118, TK-115 | 2 |
| 10 | US-116 | Daily return reminder email | 5 | Must | 8 | 6 | 4 | 3.6 | 2 | US-109, US-111 | 2 |
| 11 | US-117 | Open Loans by Equipment report | 2 | Must | 7 | 5 | 3 | 7.5 | 2 | US-109, US-118 | 2 |
| 12 | US-120 | Load open loans from the spreadsheet | 8 | Should | 6 | 6 | 5 | 2.13 | 3 | US-109, US-110, US-111, US-114, US-116, US-118, US-119, TK-115 | 3 |

Dependencies take precedence over WSJF. Small stories with a high WSJF (US-110, US-114, TK-115) are placed straight after the stories they need.

## 3. Sprint plan
| Sprint | Stories | Points | Goal |
|---|---|---|---|
| 1 | US-109, US-118, US-111, US-110, US-114, US-112, TK-115 | 19 / 20 | Reps can create valid Equipment Loans on an Account. Sales Rep, Owner and Sales Manager are set automatically. Date rules and the Shipped gate are enforced. Permission sets, manager group and sharing rule are in place. Every sales rep has a manager. |
| 2 | US-119, US-113, US-116, US-117 | 17 / 20 | Loans are private, and account owners get read access. Managers approve loans. Reps get daily return reminders. Managers have the Open Loans by Equipment report. → **MVP feature-complete** |
| 3 | US-120 (+ UAT support, defect fixing, release prep) | 8 / 20 | Spreadsheet loans are migrated: dry-run in a sandbox, then the production cut-over load with reconciliation sign-off. UAT and go-live. |

Sprint 3 has 12 points of spare capacity on purpose. It is kept for UAT defects, for rework if PO answers change the data model (R1/R2/R6), and for possible migration iterations with Lumen.

## 4. Critical path
`US-109 → US-118 → TK-115 → US-113` (approval routing needs the permission sets and managers on users)

`US-109 → US-111 → US-119 → US-120` (migration must load after the sharing flows are deployed, note #13 / R16)

The full chain to go-live is US-109 → US-111 / US-118 → US-119 (and US-116, TK-115) → US-120 dry-run → US-120 production load → go-live. Any delay to US-109, for example from open PO questions R1, R2 or R6, moves the whole plan.

## 5. Milestones
| Milestone | Target | Criteria |
|---|---|---|
| M1 – Foundation | End of sprint 1 | US-109, US-110, US-111, US-112, US-114, US-118 validated against SDO. TK-115 completed. |
| M2 – MVP feature-complete | End of sprint 2 | All Must stories QA-approved and validated (deploy validate) against SDO. |
| M3 – UAT | Sprint 3, week 1 | Lumen business users test in the sandbox. US-120 dry-run reconciled. |
| M4 – Go-live | End of sprint 3 | Release deploy to SDO, US-120 production load (before the 07:00 reminder run), bypass removed, smoke test passed. |

## 6. Risks and mitigations
| # | Risk | Impact | Mitigation |
|---|---|---|---|
| R1/R2/R6 | Status lifecycle, equipment-name type and ADR-01 not confirmed by PO | Blocks the start of US-109 and so the critical path | Get PO answers before sprint 1 starts. Picklist changes are additive. Sprint 3 buffer absorbs rework. |
| FT-105 | Migration scope and spreadsheet not confirmed | US-120 cannot run | US-120 is a Should, planned last. If it is descoped, sprint 3 becomes UAT and release only. |
| R4 | 38 active users have no manager | Approval routing fails | TK-115 in sprint 1, before US-113 |
| R9 | No org-wide email address / sender unknown | US-113 and US-116 email alerts fail to deploy | ask_human is open. Pre-deploy step creates and verifies the address before sprint 2. |
| R10 | Reminder time zone / send time | Reminder sent on the wrong day | US-116 how-to-test includes a TZ check |
| R16 | Loans loaded before US-119 is deployed | Account owners have no access to migrated loans | US-120 depends on US-119. Fallback is a Data Loader share insert. |
| R5 | Shared SDO org | Other demos affected | Additive changes only, validate-only until release |
| R13 | Email deliverability is not set to All Email in production | No approval or reminder emails | Release checklist item |
| R17 | Owner change by the flow when a manager edits a loan | Save error | Verified in US-118 AC 3. Escalate to the architect if it fails. |
| R11 (note #11) | SDO-Sales profile has no Run Reports | Reps and managers cannot run US-117 | RunReports system permission in Equipment_Loan_Manager (US-118, sprint 1) |

## 7. Assumptions
- Capacity is 20 points per 2-week sprint (team default). No other capacity information was given.
- US-119 was not in the list of items I was asked to plan, but it is in "Ready for Sprint" and US-120 depends on it, so it is planned in sprint 2.
- Migration volume is under 500 rows (US-120 assumption, not confirmed). A sandbox for the dry-run will be available in sprint 3.
- The PO's non-blocking answers on US-119 (Read access, no reassignment on account transfer) do not change the estimates. Changing them only changes the access level in the flows.
- All work is declarative (ADR-04), so the 75% Apex coverage rule only applies if code is introduced later. Validation uses `RunLocalTests` against SDO.
- Releases are deployed at the end of sprint 3 (single go-live). Sprints 1–2 are validate-only against SDO.
