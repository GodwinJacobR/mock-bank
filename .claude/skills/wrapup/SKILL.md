---
name: wrapup
description: Wrap up a task. Update OKF knowledge in knowledge/, refresh docs/ per-service diagrams and text, and update TASKS.md status.
---

# Wrapup

1. Review what changed (`git diff origin/main...HEAD`, specs touched).
2. Update `knowledge/` (OKF): add or fix entries for new or changed facts. Keep entries short.
3. Update `docs/`: docs are grouped per service in `docs/<service>/` (`gateway`, `accounts`, `ledger`, `transfers`, `web`), plus `docs/saga/` (generic saga runner) and `docs/platform/` (shared/system flow). For each service touched, update its `README.md` (role, API, data owned, events in/out, Mermaid interaction diagram) and any extra pages in that folder. Fix Mermaid diagrams and text so they match the change. Keep "Spec" vs "Proposal" labels accurate (move items to Spec once specced or built). A new service gets a new `docs/<service>/README.md`. Update `docs/README.md` (index) whenever files or folders are added, moved or removed, and fix links.
4. Update `TASKS.md`: set the task row Status (todo | in progress | done), plus Spec, Branch, PR.
5. Commit these updates on the task branch.
