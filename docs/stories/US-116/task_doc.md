# US-116: Daily return reminder email to sales rep
**Status:** Ready for UAT · **Category:** configuration · **Points:** 5 · **Sprint:** —

## Summary
Every day at 08:00 Pacific, the Sales Rep of each Shipped loan due back tomorrow gets an email reminder. Each reminder is logged as a completed email activity on the loan, so late returns can be chased in time.

## What was built
- **Flow** `Equipment_Loan_Return_Reminder`: schedule-triggered, runs daily at 08:00 org time (America/Los_Angeles). Start filter: `Status__c` = `Shipped` and `Sales_Rep__c` is not null.
- **Decision** `Is_Reminder_Due`: `Return_Date__c` = `$Flow.CurrentDate` + 1 and the Sales Rep is active.
- **Email alert** `Equipment_Loan__c.EL_Return_Reminder_Alert`: recipient = `Sales_Rep__c`, sender = `DefaultWorkflowUser`.
- **Email template** `Equipment_Loans/Equipment_Loan_Return_Reminder`: Classic Text. Subject "Equipment loan {Loan Number} is due back tomorrow". The body has the account, equipment, serial number, return date and a link to the loan.
- **Create Records** `Log_Reminder_Activity`: creates a Task with Subtype `Email`, Status `Completed`, Owner = Sales Rep, Due Date = run date, and subject "Email: Equipment loan … is due back tomorrow".
- **Email folder** `Equipment_Loans`: Public, ReadOnly. Same definition as US-113.
- No Apex. No permission set changes.

## How it works
- **AC 1:** A Shipped loan due tomorrow with an active rep → one email from the default workflow user, with a link to the loan.
- **AC 2:** Loans that are not Shipped are excluded by the start filter.
- **AC 3:** Return dates of today, in the past, or 2 or more days ahead → outcome `Not Due`, nothing is sent.
- **AC 4:** A completed email Task is created on the loan for every reminder sent.
- **AC 5:** If the rep is inactive or blank, nothing happens for that loan. Other loans are still processed.
- **AC 6:** The return date is checked when the job runs, so a moved date moves the reminder with it.
- **AC 7:** The flow is bulkified: no SOQL, and one Task insert per batch of 200.

## How to test
No unit tests. Validate with `sf project deploy validate -x docs/stories/US-116/manifest/US-116-package.xml -o SDO`; the check-only validation succeeded with 33 components. Test the ACs manually in a sandbox with Flow Builder Debug, as described in the story's How to test. Also check that Scheduled Jobs shows the next run at 08:00 Pacific.

## Deployment notes
- **Pre-deploy:**
  - US-109 and US-111 must be deployed first or in the same package.
  - Merge `workflows/Equipment_Loan__c.workflow-meta.xml` with US-113 and keep all alerts and field updates from both stories.
- **Post-deploy:**
  - Set the **Default Workflow User** and the flow error email recipient in Process Automation Settings.
  - Confirm the flow is Active and that Scheduled Jobs shows 08:00 Pacific.
  - Set Deliverability to All Email.
- No permission sets need to be assigned.

## Known limitations
- A loan set to Shipped after 08:00 with a return date of tomorrow gets no reminder.
- A manual re-run on the same day sends the reminders again. There is no "sent" flag.
- Not covered: reminders to hospital contacts, overdue escalation, SMS or Slack.
- Not tested live yet, because SDO is validate-only. Email delivery, the 200-record run and the schedule time must be checked in UAT.
- The flow name does not follow the SCH_ naming pattern (it was kept as specified).
