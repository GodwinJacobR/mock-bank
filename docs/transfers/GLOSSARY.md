# Transfers

The context that moves money between accounts.

## Language

**Transfer**:
A Customer's request to move an amount in one Currency from a Bank Account to a Beneficiary at an external bank. It reserves a Hold and ends as Completed or Failed once the external bank answers.
_Avoid_: Payment, Transaction

**Beneficiary**:
The external recipient of a Transfer, identified by its bank and account details. It is not a Bank Account.
_Avoid_: Payee, Destination account

**Pending**:
The Transfer state after submission and before an outcome: the external bank has neither confirmed settlement nor rejected it, and no Journal Entry is recorded. A Pending Transfer cannot be cancelled and ends as Completed or Failed once the external bank answers.

**Completed**:
The Transfer outcome where the external bank confirmed settlement, the Hold was Captured and the Journal Entry recorded.

**Failed**:
The Transfer outcome where the external bank rejected it, the Hold was Released and no Journal Entry was recorded.

**Bank Deposit**:
Money moved from the treasury into a Bank Account, initiated by an admin rather than a Customer. Not a Transfer.
_Avoid_: Deposit, Top-up, Funding

**Bank Withdrawal**:
Money moved from a Bank Account to the treasury, initiated by an admin rather than a Customer. Not a Transfer.
_Avoid_: Withdrawal, Cash-out
