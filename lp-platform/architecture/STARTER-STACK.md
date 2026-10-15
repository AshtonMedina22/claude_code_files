# L&P platform — starter stack

Matches [DECISIONS.md](DECISIONS.md). This is the target install, not a description of the current application repository.

Google Cloud is the host. Do not add Fly.io, Railway, Vercel, or Supabase as the place the platform runs.

## What to install

| Part | Choice | Notes |
| --- | --- | --- |
| Web application | Next.js on Cloud Run | The framework is Next.js. The host is Cloud Run, not Vercel. |
| Live database | Cloud SQL for PostgreSQL | Pursuits, prices, proposals, contracts, tasks, and the audit trail. |
| File vault | Cloud Storage | Originals, checksum, version. Never overwritten. |
| Worker | Python on Cloud Run Jobs | Parse, OCR, spreadsheet and document export, schedule checks, later connectors. Own Dockerfile so OCR libraries are in the image. |
| Waits | Cloud Workflows | Pause document intake until a person verifies, then continue. |
| Daily checks | Cloud Scheduler | Expirations, obligations, new public notices. Notify only. |
| History | BigQuery + Dataform | After live records exist. Not a second place to edit a bid. |
| Secrets and logs | Secret Manager, Cloud Logging | |
| Sign-in | Company Google accounts | The application enforces each person's permissions in Postgres. |

Add interface libraries only when a screen needs them: tables, forms, the proposal editor, a data grid, PDF viewing, charts.

## Rules

- Pin versions. Commit the lockfile.
- One application and one worker. Do not stand up a second host "because a config file exists."
- Each piece is done when the decision's proof is met (a real file reaches a verified fact; a bid reaches a saved package), not when the package installs.
