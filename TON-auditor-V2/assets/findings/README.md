# Findings

This directory holds audit output and supporting report material:

- **Reports from `/TON-auditor` runs** - written as `{project-name}-ai-audit-report-{timestamp}.md` when the skill is run with `--file-output`.
- **External audit reports** - optional third-party or manual audit `.md` files kept with the skill for operator reference.

Current orchestration does not automatically re-verify every historical report in this directory. Treat these files as saved outputs or manually supplied context unless historical finding replay is explicitly added later.
