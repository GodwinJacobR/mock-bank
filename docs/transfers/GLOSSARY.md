# Transfers

The context that moves money between accounts.

## Language

**Transfer**:
A Customer's request to move an amount in one Currency from one Bank Account to another. It reserves a Hold, results in one Journal Entry, and ends as Completed or Failed.
_Avoid_: Payment, Transaction

**Pending**:
The Transfer state after submission and before an outcome: a Hold may be reserved but no Journal Entry is recorded yet. A Pending Transfer cannot be cancelled and always ends as Completed or Failed.

**Completed**:
The Transfer outcome where the Hold was Captured and the Journal Entry recorded.

**Failed**:
The Transfer outcome where the Hold was Released and no Journal Entry was recorded.

**Deposit**:
Money moved from the treasury into a Bank Account, initiated by an admin rather than a Customer. Not a Transfer.
_Avoid_: Top-up, Funding

**Withdrawal**:
Money moved from a Bank Account to the treasury, initiated by an admin rather than a Customer. Not a Transfer.
_Avoid_: Cash-out
