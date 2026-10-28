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

### Sales Rep, Sales Manager and Owner
- The loan **Owner** and the **Sales Rep** are always the same person. If you change one, the other changes too.
- **Sales Manager** is the Sales Rep's manager, as set on their user record. It is recalculated only when the Sales Rep or the Owner changes.
- Managers and admins can change Sales Rep and Sales Manager. For sales reps, these fields are read-only.
- If you clear Sales Rep, it is set back to the current owner when you save.

> **Warning:** If your user record has no manager, the loan still saves, but Sales Manager stays blank and you cannot submit the loan for approval. Ask your administrator to set your manager.

> **Tip:** If your manager changes later, loans you already created keep the old Sales Manager. To update a loan, a manager can set the Sales Rep again.

> **Warning:** You cannot change the Account after you save the loan. If you picked the wrong account, cancel the loan and create a new one.

> **Tip:** The same serial number can appear on several loans over time.

- **Approval Status** is updated by the approval process. You cannot edit it.
- Loans are private: you see the loans you own. Managers can see all loans.
- You cannot delete an account that still has loans.

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
