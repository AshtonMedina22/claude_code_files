# Architecture history

Superseded designs, kept for reference. The current design is the owner diagram [../lp-platform-architecture.html](../lp-platform-architecture.html) with [../DECISIONS.md](../DECISIONS.md) and [../STARTER-STACK.md](../STARTER-STACK.md): everything runs on Google Cloud.

| Version | File | What it is | Why superseded |
| --- | --- | --- | --- |
| v1 | [v1-current-state-review.html](v1-current-state-review.html) | Snapshot of the CIP repo (commit `aa4512f`): layers marked built / partial / planned, plus a table of conflicts between the repo, the sheet and the old ChatGPT diagram. | Treated the CIP repo as the baseline and listed issues instead of resolving them. |
| v2 | [v2-target-fly.html](v2-target-fly.html) | First resolved target: 9 design rules, decisions D1–D10, keep / change / add relative to the repo. | Picked Fly.io from repo history. The owner chose Google Cloud and made the repo reference-only. |
| v3 | [v3-target-vercel-supabase-gcp.html](v3-target-vercel-supabase-gcp.html) | Target split across Vercel (web, MCP, workflows), Supabase (database, files, auth) and Google Cloud (workers, BigQuery). Decisions D1–D12, build phases. | Owner moved the whole platform onto Google Cloud (Cloud Run, Cloud SQL, Cloud Storage, Cloud Workflows), with no Vercel or Supabase. |

v1–v3 were also published as versions 1–3 of the same claude.ai artifact.
