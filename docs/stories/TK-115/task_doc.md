# TK-115: Verify and populate Manager on all sales rep users
**Status:** Ready for UAT (cancelled by PO) · **Category:** script · **Points:** 2 · **Sprint:** n/a

## Summary
The product owner cancelled this task. SDO is a test org, so existing users stay as they are, and the admin keeps the Manager field on user records up to date outside this project. Nothing was built or changed.

## What was built
- Nothing: no metadata, no scripts, no queries, no user updates.
- The ticket's Out of Scope, Solution Components, How to Test and Manual Steps fields now record the cancellation.

## How it works
- AC 1–4 (gap report, managers populated, managers active and authorised, evidence): not applicable because the task was cancelled.
- Delete rule confirmed by the PO: only admins may delete loans. The US-118 permission sets already set allowDelete=false, so nothing needed to change.

## How to test
Not applicable. Risk for testers of US-111, US-113 and US-120: if a rep's user record has no Manager, their loans get an empty Sales Manager, and approvals can't be routed. Use test users who have a Manager set by the admin.

## Deployment notes
None. The admin maintains User.ManagerId outside the project. At the US-118 release, check that both permission sets block delete.

## Known limitations
- Nothing checks that reps have an active manager who holds Equipment_Loan_Manager.
- Changing a user's Manager doesn't update Sales Manager on existing loans. To refresh it, an admin re-saves the loan's Sales Rep.
- The planner still needs to remove TK-115 from US-120's dependencies.
