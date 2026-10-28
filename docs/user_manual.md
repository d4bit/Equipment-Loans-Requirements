# User manual

## How to record an equipment loan

An **Equipment Loan** records a demo device lent to a hospital. Each loan gets a **Loan Number** (for example EL-00001).

1. Create a new Equipment Loan.
2. Pick the **Account** (the hospital). Only business accounts can be chosen.
3. Choose the **Equipment Name** from the list (Ultrasound U1, Patient Monitor M5, Blood Analyzer A200, Infusion Pump P3, ECG Device E12).
4. Enter the **Serial Number**, **Start Date** and **Return Date**. All four fields are required.
5. Save. The loan starts with **Status** = Requested and **Approval Status** = Not Submitted.

**Status values:** Requested, Approved, Shipped, Returned, Rejected, Cancelled. Requested, Approved and Shipped count as open loans.

> **Warning:** You cannot change the Account after you save the loan. If you picked the wrong account, cancel the loan and create a new one.

> **Tip:** The same serial number can appear on several loans over time.

- **Approval Status** is updated by the approval process. You cannot edit it.
- Loans are private: you see the loans you own. Managers can see all loans.
- You cannot delete an account that still has loans.
