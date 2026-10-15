# Architecture history

The current design is v3: [../lp-platform-architecture.html](../lp-platform-architecture.html) with [../DECISIONS.md](../DECISIONS.md).

| Version | File | What it is | Status |
| --- | --- | --- | --- |
| v1 | [v1-current-state-review.html](v1-current-state-review.html) | Snapshot of the CIP repo (commit `aa4512f`): layers marked built / partial / planned, plus a table of conflicts between the repo, the sheet and the old ChatGPT diagram. | Superseded. It treated the CIP repo as the baseline and listed issues instead of resolving them. |
| v2 | [v2-target-fly.html](v2-target-fly.html) | First resolved target: 9 design rules, decisions D1–D10, keep / change / add relative to the repo. | Superseded. It picked Fly.io; the owner chose Google Cloud and made the repo reference-only. |
| Cursor alternative | [cursor-all-gcp/](cursor-all-gcp/DECISIONS.md) | Everything on Google Cloud: Cloud Run web, Cloud SQL, Cloud Storage, Cloud Workflows, Google sign-in. | **Unconfirmed.** A Cursor agent pushed it to `main` (commit `6085989`, 14 Oct 2026) without the owner's request. It was reverted, and is kept here as an option for the open hosting decision. |

v1–v3 were also published as versions 1–3 of the same claude.ai artifact.
