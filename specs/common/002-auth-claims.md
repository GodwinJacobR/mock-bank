# Auth Claims

Status: draft

## Goal
Library crate `common/auth` (`bank-auth`): the internal signed JWT the gateway sends to services, and helpers to mint and verify it.

## Scope (in)
- `Claims` type, sign and verify helpers.
- Axum extractor yielding the authenticated caller (`Caller { customer_id, role }`).
- Role check helpers.

## Out of scope
- User login, passwords, sessions (accounts service).
- Public session cookies and admin token handling (gateway).

## Design
- `Claims { sub: Uuid (customer id), iss: "gateway", exp, iat, role: Role }`, `Role = customer | admin`.
- Algorithm HS256, secret from env `JWT_SECRET`. Gateway mints with a 60 s lifetime; verification allows 5 s clock leeway and requires `iss = gateway`.
- Header contract: `Authorization: Bearer <jwt>` plus `X-Customer-Id: <uuid>`. The extractor rejects the request (`401`) if the header and `sub` disagree.
- Admin / service token role: when the gateway sees a valid `X-Admin-Token` (equal to `ADMIN_TOKEN`) it mints a token with `role = admin` (the admin/service role). `sub` is the customer being acted for, or the nil UUID when none. Services honour `admin` for privileged operations, e.g. ledger allows treasury-origin transfers only for it. The admin token itself never leaves the gateway.
- Helpers: `Caller::require_admin()` (else `forbidden`), `Caller::owns(customer_id)`.
- Errors map to `AppError::unauthorized` / `forbidden` from `common/errors`.

## Dependencies
- `common/001-money-and-errors`.

## Acceptance criteria
- [ ] Token signed in one crate verifies in another.
- [ ] Expired, tampered, wrong-issuer, or wrong-secret tokens are rejected with `unauthorized`.
- [ ] Mismatch between `X-Customer-Id` and `sub` is rejected.
- [ ] `require_admin` rejects a `customer` role with `forbidden`.

## Test plan
- Unit tests for sign/verify round trip, expiry (with controllable clock), tampering, header mismatch, role checks.
