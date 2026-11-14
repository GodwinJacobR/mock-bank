# Ledger

The context that records postings against accounts as the book of record for money.

## Language

**Ledger Account**:
A place where postings accumulate, including non-customer accounts such as the treasury. Linked to a Bank Account only by ID.
_Avoid_: Account

**Currency**:
The single unit of money a Ledger Account is denominated in. All Postings on a Ledger Account, and all Postings in a Journal Entry, share one Currency.
_Avoid_: Denomination

**Journal Entry**:
A balanced, immutable record made up of two or more Postings, where total debits equal total credits. A Transfer results in one Journal Entry.
_Avoid_: Transaction, Entry

**Posting**:
One debit or credit line against a single Ledger Account, belonging to exactly one Journal Entry.
_Avoid_: Entry, Line

**Balance**:
The net of all Postings on a Ledger Account (credits minus debits for a liability account such as a customer's). Always derived from Postings, never an independently stored fact.
_Avoid_: Current amount, Funds

**Hold**:
A reservation of funds against a Ledger Account, made before a Transfer settles. It reduces Available Balance but is not money movement and is not a Posting.
_Avoid_: Lock, Authorization

**Captured**:
The Hold outcome where the reservation is converted into the Journal Entry that settles the Transfer.

**Released**:
The Hold outcome where the reservation is explicitly cancelled and the funds become available again.

**Available Balance**:
The Balance of a Ledger Account minus its active Holds. The amount that can still be committed.
_Avoid_: Spendable, Free balance
