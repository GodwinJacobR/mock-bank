# Specs

| Spec | Title | Depends on |
|---|---|---|
| [000](000-architecture.md) | Architecture | none |
| [001a](001a-ci.md) | CI | none |
| [001b](001b-workspace-compose.md) | Workspace and Compose | none |
| [001c](001c-accounts-service.md) | Accounts service | 001b |
| [001d](001d-ledger-service.md) | Ledger service | 001b |
| [001e](001e-gateway.md) | Gateway | 001b, 001c, 001d |
| [001f](001f-web-ui.md) | Web UI | 001b, 001e |
| [001g](001g-seed.md) | Seed | 001e |

## Build order
1. 001a and 001b first.
2. 001c and 001d in parallel.
3. 001e.
4. 001f and 001g in parallel.
