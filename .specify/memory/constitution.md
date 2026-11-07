# Mock Bank Constitution

## Core Principles

### I. Rust
All services and libraries are written in Rust.

### II. Spec First (NON-NEGOTIABLE)
No code without an approved spec. Implement only what the spec says; no extra scope.

### III. Feature Branch and PR for Every Change
Every change lands through a feature branch and a pull request. Never commit directly to main.

### IV. Synthetic Data Only
All data is synthetic. No real customer, account, or personal data anywhere in code, tests, fixtures, or docs.

### V. No Service-to-Service HTTP
Services never call each other over HTTP. Integration happens through the mechanisms defined in specs/000-architecture.md.

### VI. Money as i64 Minor Units
All monetary amounts are i64 minor units (for example cents). No floats for money.

## Governance
This constitution overrides other practices. Amendments require a PR that updates this file.

**Version**: 1.0.0 | **Ratified**: 2026-11-07 | **Last Amended**: 2026-11-07
