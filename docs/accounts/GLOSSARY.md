# Accounts

The context that owns customers' bank accounts and their lifecycle.

## Language

**Bank Account**:
A customer's account, with exactly one owner (a Customer), a status, and a product.
_Avoid_: Account

**Customer**:
A person who owns Bank Accounts. Each Bank Account has exactly one Customer.
_Avoid_: User, Client

**Product**:
A label on a Bank Account naming its kind (for example checking or savings). It carries no rules.
_Avoid_: Account type, Plan

**Open**:
The Bank Account status in which it can send and receive Transfers.

**Frozen**:
The Bank Account status, set by an admin, in which it can neither send nor receive Transfers.

**Closed**:
The terminal Bank Account status. A Bank Account can only be Closed when its Balance is zero and it has no active Holds.
