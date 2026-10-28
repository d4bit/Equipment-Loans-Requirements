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
