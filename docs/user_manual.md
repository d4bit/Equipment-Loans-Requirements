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

Every loan must be approved by your manager (the loan's **Sales Manager**) before you can set its **Status** to **Shipped**. This rule applies to everyone, including managers and administrators.

**To submit a loan for approval:**
1. Open a loan with **Status** = Requested. **Sales Manager** must be filled in.
2. Click **Submit for Approval** and add a comment if you want.
3. **Approval Status** changes to **Pending** and your manager receives the email "Equipment loan EL-xxxxx needs your approval". While the request is pending, you cannot edit the loan.

**For managers – to approve or reject:**
1. Open the link in the email, or open the loan and go to the **Approval History** list.
2. Click **Approve** or **Reject**. When you reject, add a comment explaining why. The rep receives your comment.

**What happens next:**
- **Approved:** **Approval Status** = Approved and **Status** stays Requested. You get the email "Equipment loan EL-xxxxx approved". Open the loan, set **Status** to **Shipped** and save.
- **Rejected:** **Approval Status** = Rejected. You get the email "Equipment loan EL-xxxxx rejected" with your manager's comment. Fix the loan and click **Submit for Approval** again.
- **Recall:** to withdraw a pending request, click **Recall** in **Approval History**. **Approval Status** goes back to Not Submitted and you can edit the loan again.

**What you may see:**
- "…does not meet the entry criteria or initial submitters of any active approval processes…" appears if the loan is already Pending or Approved, is not in Status Requested, or has no **Sales Manager**. If your loan has no Sales Manager, ask your administrator to set your manager on your user record.
- "This loan must be approved by your manager before it can be shipped." appears if you try to set Shipped before approval, or if you try to create a new loan directly as Shipped.
- "Approval Status is set by the approval process and cannot be changed manually." appears if you try to edit **Approval Status** yourself.

**Tips:**
- Your manager's delegated approver cannot approve loans. Only your own manager can.
- An approved loan does not need to be approved again. If dates or equipment must change after approval, ask an administrator.

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
