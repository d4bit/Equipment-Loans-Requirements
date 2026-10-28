# User manual

## How to record an equipment loan

An **Equipment Loan** records a demo device lent to a hospital. Each loan gets a **Loan Number** (for example EL-00001).

1. Create a new Equipment Loan.
2. Pick the **Account** (the hospital). Only business accounts can be chosen.
3. Choose the **Equipment Name** from the list (Ultrasound U1, Patient Monitor M5, Blood Analyzer A200, Infusion Pump P3, ECG Device E12).
4. Enter the **Serial Number**, **Start Date** and **Return Date**. All four fields are required.
5. Leave **Sales Rep** blank. When you save, it is filled in with your name.
6. Save. The loan starts with **Status** = Requested and **Approval Status** = Not Submitted. **Sales Manager** is filled in with your manager.

**Status values:** Requested, Approved, Shipped, Returned, Rejected, Cancelled. Requested, Approved and Shipped count as open loans.

### Loan dates: the 30-day rule
A demo loan can last at most 30 days.
- **Return Date** must be after **Start Date**. If they are the same day or the return is earlier, you see: *"Return Date must be after Start Date."*
- **Return Date** can be at most 30 days after **Start Date**. For example, a loan starting 1 March may be returned on 31 March at the latest. If you go past that, you see: *"Return Date must be no more than 30 days after Start Date."*
- A start date in the past is allowed.

> **Note:** The rule also applies when you edit a loan, whichever date you change. It applies to everyone, including administrators and data imports.

## Who can do what with equipment loans

What you can do depends on the access your administrator gives you.

| Action | Sales rep | Sales manager |
|---|---|---|
| Create a loan and edit its details (Equipment Name, dates, Status) | Yes | Yes |
| See every loan in the company | Only the loans you have access to | Yes |
| Change **Sales Rep** or **Sales Manager** | No (read-only) | Yes |
| Change **Approval Status** | No | No (the approval process updates it) |
| Delete a loan | No | No |

**Tips**
- If you can't find the **Equipment Loans** tab, or the Equipment Loans list on an account, you don't have access yet. Ask your Salesforce administrator.
- When a manager changes the **Sales Rep**, the loan's owner changes to that person automatically.

## Shipping a loan: manager approval required

Before a loan can be shipped, the sales rep's manager has to approve it. The approval goes to the person shown in **Sales Manager** on the loan. That field is filled from the Manager on the sales rep's user record when the loan is created or when its Sales Rep changes.

> **Warning:** If **Sales Manager** is empty, the approval has no one to go to. Ask your administrator to set your Manager on your user record. The administrator then re-saves the loan's Sales Rep to fill the field.

> **Note:** Only administrators can delete loans.

## Finding and opening equipment loans

**Open the Equipment Loans tab**
1. Click the App Launcher (the grid icon, top left).
2. Search for **Equipment Loans** and open it. The tab is not in any app menu bar. If you can't find it, ask your admin for the Equipment Loan permission set.
3. Pick a list view:
   - **Open Loans**: every loan you can see with Status Requested, Approved or Shipped.
   - **My Loans**: loans you own, which means loans where you are the Sales Rep.

**Create a loan from a hospital account**
1. Open the account.
2. In the **Equipment Loans** related list, click **New**. The Account is already filled in.
3. Fill in Equipment Name, Serial Number, Start Date and Return Date, then click **Save**.

The related list shows Loan Number, Equipment Name, Serial Number, Start Date, Return Date and Status, with the latest return dates first.

**Reading a loan record**
- The top panel shows Loan Number, Account, Equipment Name, Status and Return Date.
- The **path** below the panel shows the loan stage: Requested → Approved → Shipped → Returned, plus Rejected and Cancelled. Rejected and Cancelled appear as steps on the path even though they are end states.
- **Loan Details** holds the loan data. **Approval** shows Approval Status, Sales Rep and Sales Manager.
- Related lists: **Approval History**, **Activity History** and **Equipment Loan History** (field changes).

> **Note:** Approval Status can never be edited by hand. Reps can't edit Sales Rep or Sales Manager either. Loans have no Change Owner action.

## Return reminder emails

If you are the **Sales Rep** on a loan, you get an email reminder the day before the equipment is due back.

**When you get it**
- Every day at 08:00 Pacific time, including weekends and holidays.
- Only for loans with Status **Shipped** whose **Return Date** is tomorrow.

**What it contains**
- Subject: "Equipment loan {Loan Number} is due back tomorrow".
- The account, equipment, serial number and return date.
- A link that opens the loan.

**What to do**
1. Open the loan from the link.
2. Contact the hospital so the equipment is returned on time.

**Where to find past reminders:** each reminder appears on the loan's **Activity** timeline as a completed email activity, assigned to you.

> **Tip:** If the return date changes, the reminder moves with it. You get the reminder the day before the new date.

> **Note:** You get no reminder if:
> - the loan is not Shipped,
> - it was shipped after 08:00 on the day before it is due,
> - no Sales Rep is set on the loan, or
> - the Sales Rep's user is inactive.
