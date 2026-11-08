---
name: wrapup
description: Wrap up a task. Update OKF knowledge in knowledge/, refresh docs/ diagrams and text, and update TASKS.md status.
---

# Wrapup

1. Review what changed (`git diff origin/main...HEAD`, specs touched).
2. Update `knowledge/` (OKF): add or fix entries for new or changed facts. Keep entries short.
3. Update `docs/`: fix Mermaid diagrams and text so they match the change. Keep "Spec" vs "Proposal" labels accurate (move items to Spec once specced or built). Update `docs/README.md` if files were added.
4. Update `TASKS.md`: set the task row Status (todo | in progress | done), plus Spec, Branch, PR.
5. Commit these updates on the task branch.
