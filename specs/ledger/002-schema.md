# Ledger DB Schema

Status: draft

## Goal
Full Postgres DDL, grants and invariant enforcement for schema `ledger`.

## Scope (in)
- Tables: `account_projection`, `balances`, `transfers`, `entries`, `idempotency_keys`, `processed_events`, `outbox`.
- Constraints, indexes, grants per DB role, append-only entries, zero-sum enforcement, treasury accounts.
- Migration naming.

## Out of scope
- Customers, payees, sessions (`accounts/002-schema`).
- FX and cross-currency.

## Design
**Migrations**: `migrations/ledger/NNNN_description.sql` (4-digit zero-padded, increasing; never edited after merge). Phase 1: `0001_init.sql` (tables), `0002_invariants.sql` (triggers), `0003_treasury.sql` (seed treasury rows), `0004_grants.sql`. Run by `ledger_owner` at service start.

**Roles**: `ledger_owner` owns the schema and objects (migrations only). `ledger_svc` is the runtime role.

```mermaid
erDiagram
  account_projection ||--|| balances : has
  account_projection ||--o{ entries : posts_to
  transfers ||--|{ entries : has
  account_projection ||--o{ transfers : debited_from
  account_projection ||--o{ transfers : credited_to
  account_projection {
    uuid account_id PK
    uuid customer_id "null for treasury"
    text account_number UK
    char3 currency
    text status
    boolean is_treasury
    timestamptz updated_at
  }
  balances {
    uuid account_id PK,FK
    char3 currency
    bigint balance_minor
    timestamptz updated_at
  }
  transfers {
    uuid id PK
    uuid from_account FK
    uuid to_account FK
    bigint amount_minor
    char3 currency
    text memo
    text status
    timestamptz created_at
  }
  entries {
    bigint id PK
    uuid transfer_id FK
    uuid account_id FK
    bigint amount_minor
    timestamptz created_at
  }
  idempotency_keys {
    text key PK
    uuid customer_id PK
    text request_hash
    jsonb response
    int status_code
    timestamptz created_at
  }
  processed_events {
    uuid event_id PK
    text consumer PK
    timestamptz processed_at
  }
  outbox {
    uuid id PK
    text topic
    text key
    jsonb payload
    timestamptz published_at
  }
```
`idempotency_keys.customer_id` holds the caller's customer id (the nil UUID for admin calls without a customer). `processed_events` and `outbox` are standalone.

**DDL** (`0001_init.sql`)
```sql
CREATE SCHEMA IF NOT EXISTS ledger AUTHORIZATION ledger_owner;
SET search_path = ledger;

CREATE TABLE account_projection (
  account_id     uuid PRIMARY KEY,               -- same id as accounts.accounts.id (or generated for treasury)
  customer_id    uuid,                           -- NULL only for treasury
  account_number text        NOT NULL,           -- 12 digits, or 'bank:treasury:<CUR>'
  currency       char(3)     NOT NULL,
  status         text        NOT NULL DEFAULT 'open',
  is_treasury    boolean     NOT NULL DEFAULT false,
  updated_at     timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT proj_number_uk UNIQUE (account_number),
  CONSTRAINT proj_currency_fmt CHECK (currency ~ '^[A-Z]{3}$'),
  CONSTRAINT proj_status_chk CHECK (status IN ('open','closed')),
  CONSTRAINT proj_treasury_chk CHECK (
    (is_treasury AND customer_id IS NULL AND account_number = 'bank:treasury:' || currency)
    OR (NOT is_treasury AND customer_id IS NOT NULL AND account_number ~ '^[0-9]{12}$'))
);
CREATE INDEX proj_customer_idx ON account_projection (customer_id);

CREATE TABLE balances (
  account_id    uuid PRIMARY KEY REFERENCES account_projection(account_id),
  currency      char(3) NOT NULL,
  balance_minor bigint  NOT NULL DEFAULT 0,
  updated_at    timestamptz NOT NULL DEFAULT now()
);
-- Overdraft guard: only treasury may be negative. Enforced in the DB via trigger in 0002.

CREATE TABLE transfers (
  id           uuid PRIMARY KEY,
  from_account uuid        NOT NULL REFERENCES account_projection(account_id),
  to_account   uuid        NOT NULL REFERENCES account_projection(account_id),
  amount_minor bigint      NOT NULL,
  currency     char(3)     NOT NULL,
  memo         text        NOT NULL DEFAULT '',
  status       text        NOT NULL DEFAULT 'posted',
  created_at   timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT transfers_amount_pos CHECK (amount_minor > 0),
  CONSTRAINT transfers_distinct CHECK (from_account <> to_account),
  CONSTRAINT transfers_status_chk CHECK (status IN ('posted')),
  CONSTRAINT transfers_memo_len CHECK (char_length(memo) <= 140)
);
CREATE INDEX transfers_from_idx ON transfers (from_account, created_at DESC, id DESC);
CREATE INDEX transfers_to_idx   ON transfers (to_account,   created_at DESC, id DESC);

CREATE TABLE entries (
  id           bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  transfer_id  uuid        NOT NULL REFERENCES transfers(id),
  account_id   uuid        NOT NULL REFERENCES account_projection(account_id),
  amount_minor bigint      NOT NULL,             -- signed: debit negative, credit positive
  created_at   timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT entries_nonzero CHECK (amount_minor <> 0),
  CONSTRAINT entries_one_per_account UNIQUE (transfer_id, account_id)
);
CREATE INDEX entries_account_idx ON entries (account_id, id DESC);

CREATE TABLE idempotency_keys (
  key          text        NOT NULL,
  customer_id  uuid        NOT NULL,
  request_hash text        NOT NULL,             -- sha256 hex of canonical request
  response     jsonb       NOT NULL,
  status_code  integer     NOT NULL,
  created_at   timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (key, customer_id)
);

CREATE TABLE processed_events (                  -- used by common/events consumer
  event_id     uuid        NOT NULL,
  consumer     text        NOT NULL,
  processed_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (event_id, consumer)
);

-- standard outbox from common/005-outbox
CREATE TABLE outbox (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  topic        text        NOT NULL,
  key          text        NOT NULL,
  payload      jsonb       NOT NULL,
  created_at   timestamptz NOT NULL DEFAULT now(),
  published_at timestamptz,
  attempts     integer     NOT NULL DEFAULT 0,
  last_error   text
);
CREATE INDEX outbox_unpublished_idx ON outbox (created_at) WHERE published_at IS NULL;
```

**Invariants** (`0002_invariants.sql`)
1. Append-only entries and transfers:
```sql
CREATE FUNCTION forbid_mutation() RETURNS trigger LANGUAGE plpgsql AS
$$ BEGIN RAISE EXCEPTION '% on % is not allowed (append-only)', TG_OP, TG_TABLE_NAME; END $$;
CREATE TRIGGER entries_append_only   BEFORE UPDATE OR DELETE ON entries   FOR EACH ROW EXECUTE FUNCTION forbid_mutation();
CREATE TRIGGER transfers_append_only BEFORE UPDATE OR DELETE ON transfers FOR EACH ROW EXECUTE FUNCTION forbid_mutation();
CREATE TRIGGER entries_no_truncate   BEFORE TRUNCATE ON entries   FOR EACH STATEMENT EXECUTE FUNCTION forbid_mutation();
CREATE TRIGGER transfers_no_truncate BEFORE TRUNCATE ON transfers FOR EACH STATEMENT EXECUTE FUNCTION forbid_mutation();
```
Defence in depth: the grants (below) also give `ledger_svc` no UPDATE/DELETE/TRUNCATE on these tables, and `ledger_svc` is not the owner, so it cannot drop the triggers.

2. Zero-sum per transfer: a deferred constraint trigger checks at commit.
```sql
CREATE FUNCTION check_transfer_zero_sum() RETURNS trigger LANGUAGE plpgsql AS $$
DECLARE s bigint; n integer;
BEGIN
  SELECT coalesce(sum(amount_minor),0), count(*) INTO s, n FROM entries WHERE transfer_id = NEW.transfer_id;
  IF s <> 0 OR n <> 2 THEN
    RAISE EXCEPTION 'transfer % must have exactly 2 entries summing to 0 (sum=%, n=%)', NEW.transfer_id, s, n;
  END IF;
  RETURN NULL;
END $$;
CREATE CONSTRAINT TRIGGER entries_zero_sum AFTER INSERT ON entries
  DEFERRABLE INITIALLY DEFERRED FOR EACH ROW EXECUTE FUNCTION check_transfer_zero_sum();
```
Entries for a transfer are inserted in the same transaction as the transfer, so the check runs at commit and sees both rows. Entry currency matching the transfer is enforced by the service; the audit view re-checks it.

3. Overdraft guard: only treasury may go negative.
```sql
CREATE FUNCTION check_balance_nonneg() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  IF NEW.balance_minor < 0 AND NOT (SELECT is_treasury FROM account_projection WHERE account_id = NEW.account_id) THEN
    RAISE EXCEPTION 'insufficient_funds for account %', NEW.account_id;
  END IF;
  RETURN NEW;
END $$;
CREATE TRIGGER balances_nonneg BEFORE INSERT OR UPDATE ON balances FOR EACH ROW EXECUTE FUNCTION check_balance_nonneg();
```
The service checks funds first and returns `422 insufficient_funds`; the trigger is the backstop.

4. Audit views (read-only, used by tests and a periodic audit):
```sql
CREATE VIEW audit_zero_sum_violations AS
  SELECT t.id FROM transfers t LEFT JOIN entries e ON e.transfer_id = t.id
  GROUP BY t.id HAVING coalesce(sum(e.amount_minor),0) <> 0 OR count(e.id) <> 2;
CREATE VIEW audit_balance_mismatches AS
  SELECT b.account_id FROM balances b
  LEFT JOIN (SELECT account_id, sum(amount_minor) s FROM entries GROUP BY account_id) x USING (account_id)
  WHERE b.balance_minor <> coalesce(x.s,0);
```
Both must return zero rows.

**Treasury accounts** (`0003_treasury.sql`): inserted by migration, not by events.
```sql
INSERT INTO account_projection (account_id, customer_id, account_number, currency, is_treasury)
SELECT gen_random_uuid(), NULL, 'bank:treasury:' || c, c, true FROM unnest(ARRAY['USD','EUR','GBP']) AS c
ON CONFLICT (account_number) DO NOTHING;
INSERT INTO balances (account_id, currency) SELECT account_id, currency FROM account_projection WHERE is_treasury
ON CONFLICT DO NOTHING;
```
Treasury has no counterpart in the accounts service and never appears in `accounts.events.v1`. Rebuilding the projection (truncate non-treasury rows, replay) must keep `is_treasury` rows. Origination from a treasury source requires the admin role, checked in the service.

**Projection replay**: `account_projection` and `balances` rows for non-treasury accounts are created by the `AccountOpened` handler (`INSERT ... ON CONFLICT (account_id) DO UPDATE` for projection; `balances ... ON CONFLICT DO NOTHING` so balances derived from entries survive a projection replay). A projection reset therefore only deletes projection rows that have no entries, or replays with upsert only; it never deletes `entries`.

**Grants** (`0004_grants.sql`)
```sql
REVOKE ALL ON SCHEMA ledger FROM PUBLIC;
GRANT USAGE ON SCHEMA ledger TO ledger_svc;
GRANT SELECT, INSERT, UPDATE ON account_projection, balances, outbox TO ledger_svc;
GRANT SELECT, INSERT         ON transfers, entries, idempotency_keys   TO ledger_svc;  -- append-only
GRANT SELECT, INSERT         ON processed_events                      TO ledger_svc;
GRANT DELETE                 ON processed_events                      TO ledger_svc;  -- consumer reset helper
GRANT USAGE ON SEQUENCE entries_id_seq TO ledger_svc;
GRANT SELECT ON audit_zero_sum_violations, audit_balance_mismatches   TO ledger_svc;
-- no DELETE/TRUNCATE on entries, transfers; no DDL; no access to schema accounts
```

## Dependencies
- `platform/002-workspace-and-compose` (roles), `common/005-outbox` (outbox DDL), `common/004-events-kafka` (`processed_events` shape).

## Acceptance criteria
- [ ] Migrations apply on an empty database as `ledger_owner`; treasury rows exist for each configured currency.
- [ ] `UPDATE`/`DELETE`/`TRUNCATE` on `entries` and `transfers` fail for `ledger_svc` (privilege) and for the owner (trigger).
- [ ] Committing a transfer whose entries do not sum to zero, or are not exactly two, fails at commit.
- [ ] A negative balance on a non-treasury account is rejected; treasury may go negative.
- [ ] Both audit views return zero rows after a mixed workload.
- [ ] Duplicate `(key, customer_id)` idempotency insert is rejected; duplicate `(event_id, consumer)` is rejected.
- [ ] `ledger_svc` cannot access schema `accounts` or run DDL.

## Test plan
- Integration (testcontainers Postgres): apply migrations; attempt each violating statement as `ledger_svc` and as `ledger_owner`; property-style test inserting random valid transfers then asserting the audit views are empty.
