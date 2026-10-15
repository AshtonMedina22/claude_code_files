# L&P Platform — Architecture Decisions

Target architecture v3 · 14 Oct 2026 · Visual: [lp-platform-architecture.html](lp-platform-architecture.html)

**Sources of truth:** the Reference sheet (Overview, Overview Map, Features, Fixes) and the owner's platform decisions. The existing CIP repository (`AshtonMedina22/Contract-Intelligence-Platform`) is reference material only. Reuse vs. rebuild is decided per component.

## What it is

An internal **business development (BD) lifecycle platform** for a security services contractor. It covers capture, proposal and pricing management; win/loss and competitive intelligence; sales prospecting; client success; and contract administration after award. Staff and their AI agents (ChatGPT, Claude) use it through one web app and one MCP server.

Avoid the name "Contract Intelligence Platform". That term means AI analysis of signed contracts, which is only the post-award module.

## Status

**Decided by the owner, 14 Oct 2026:** v3 is the platform architecture. Google Cloud runs compute and analytics; Vercel hosts the web app, API and MCP server; Supabase holds the database, files and sign-in. An all-Google-Cloud alternative (written by a Cursor agent, not adopted) is kept in [history/cursor-all-gcp/](history/cursor-all-gcp/DECISIONS.md) for reference. Owner-facing summary: [L&P Platform Tech Stack](https://claude.ai/code/artifact/ea5ffbd5-4c22-4601-89c9-5c177a9d9faa).

## Platform assignments

| Platform | Hosts |
| --- | --- |
| Vercel | Next.js web app, API, MCP server, Vercel Workflow (coordination) |
| Supabase | Postgres system of record, Storage evidence vault, Auth (staff sign in with L&P Google accounts) + OAuth 2.1 server for agents |
| Google Cloud | Cloud Run services and jobs (all background compute), BigQuery, Dataform, Artifact Registry, Secret Manager, Cloud Scheduler, Cloud Logging/Monitoring |
| Cross-cloud | Vercel → GCP uses Workload Identity Federation (OIDC). No shared secrets. |

## Design rules

1. **One operational system of record.** Supabase Postgres + Storage. BigQuery is an analytical copy. Drive, Docs and Sheets are sources or outputs.
2. **One business-logic layer.** The UI, MCP tools, workflows and jobs all call the same domain services.
3. **One agent interface.** A single remote MCP server. Agents use per-user OAuth and get that person's permissions.
4. **Two ingestion paths.** Documents go through staging, then human verification, then canonical. Authoritative system records (QuickBooks, Belfry, Apollo) sync directly with source IDs, and only mismatches go to review.
5. **Core works with no AI keys.** Semantic search, cloud OCR and in-app Ask are optional add-ons with fallbacks.
6. **Vercel coordinates, Google Cloud executes.** Heavy and long-running work runs on Cloud Run.
7. **Workflows only where work waits** on a person or an outside system.
8. **Metrics defined once.** Postgres runs live metrics; Dataform models use the same definitions and must reconcile with them.
9. **Internal users only.** Buyers, prospects, clients and competitors are records. No portals.
10. **Stay portable to Google Cloud.** Use Supabase for sign-in and file storage only. Keep business rules in the platform's own code and plain PostgreSQL, never in Supabase-only features, so the database can move to Cloud SQL as a planned migration.

## Layers

| # | Layer | Runs on | Contents |
| --- | --- | --- | --- |
| 1 | Presentation | Vercel | Next.js UI: Home & My work, Pursuits, Proposals, Pricing, Intelligence, Sales, Clients, Contracts, Data Ops |
| 2 | API & agent gateway | Vercel Functions | Server Actions/Route Handlers (Zod); MCP server (TS SDK, spec 2026-07-28); identity & access (Supabase Auth, OAuth 2.1, RLS, audit) |
| 3 | Domain services | Vercel (TypeScript) | capture, proposal, pricing, intel, contracts, sales, clients, search/metrics/reports, work management, data ops |
| 4 | AI add-ons (optional) | Model APIs | LLM gateway (Vercel AI SDK), embeddings, Google Document AI for hard scans |
| 5 | Orchestration | Vercel Workflow | Durable workflows (document lifecycle, proposal review, records-request follow-up, renewals); job dispatch to Cloud Run via WIF |
| 6 | Processing & integration | Google Cloud Run | Parsing/OCR, extraction & validation, web acquisition, solvers/ML, connector jobs, warehouse loader |
| 7 | Operational data | Supabase | Evidence vault, staging, canonical records, synced source records, work state, retrieval index (FTS + pgvector), audit & lineage |
| 8 | Analytical data | Google BigQuery | Raw → curated → reporting datasets; Dataform models; results served through the API |

## Decisions

| ID | Decision | What it means to build |
| --- | --- | --- |
| D1 | Google Cloud is the compute and analytics cloud | One GCP project per environment; images in Artifact Registry; no other worker hosts |
| D2 | Vercel hosts web/API/MCP; Supabase is the system of record | Next.js on Vercel; Supabase with RLS from day one |
| D3 | Vercel → GCP via Workload Identity Federation | Workload identity pool; one service account per Cloud Run service; Secret Manager for third-party keys |
| D4 | Sales and client success are internal modules | Sales and Clients domains with their own tables/services; contacts are records |
| D5 | One MCP server on the official TS SDK, spec 2026-07-28; no parallel REST "actions" API | Tool registry wrapping domain services; `readOnlyHint` on read tools |
| D6 | Agents sign in per user via Supabase OAuth 2.1 | Enable OAuth server; consent page; tokens map to UI roles |
| D7 | Write approval = client confirmation + server permissions | Human-only gates (verify, final price, submit) enforced as permissions in domain services |
| D8 | Core runs with no model keys | Open-source OCR default; FTS when embeddings off; agents reason via MCP |
| D9 | Connectors sync authoritative data directly | Cloud Run Job per connector; synced tables with source ID, sync log, reconciliation exceptions |
| D10 | Work management is a platform service | Tasks, checkpoints, handoffs + MCP tools to create tasks, save checkpoints, list next actions |
| D11 | Single organization with org-scoped data model | `organization_id` + RLS on every tenant table |
| D12 | BigQuery for history/cross-system; Postgres for live metrics | Incremental loader with snapshots; Dataform reconciled to operational totals |

## Data paths

- **Documents:** preserve original (vault) → extract to staging (Cloud Run) → **human verification** → promote to canonical → serve (search, drafting, metrics, MCP)
- **System records:** pull (Cloud Run Job) → upsert with source ID → reconcile mismatches → link to contract/site/period → serve
- **Analytics:** incremental load with snapshots → BigQuery raw → Dataform curated → reconcile to app totals → serve via API/MCP

Trust controls: verification state (`AI_EXTRACTED`, `NEEDS_REVIEW`, `HUMAN_VERIFIED`, `REJECTED`, `CONFLICT`). Data classification by purpose. Commercial truths kept separate: requested, proposed, awarded, current, actual.

## Build phases

1. **Foundation.** Projects + WIF; auth/roles/RLS/audit; vault → staging → verification on Cloud Run; correct pricing classification and paid vs. billable hours. *Done when a real RFP goes from upload to verified facts.*
2. **One complete bid.** Compliance matrix → staffing & pricing → authoring → DOCX/Docs export; review workflow; work management. *Done when there's a saved review package with a real blocker list.*
3. **Agents.** MCP + OAuth; read/write tools for phase 1–2 workflows; test ChatGPT and Claude read → write → resume.
4. **Revenue data.** QuickBooks + Belfry connectors; contract administration; billing reconciliation; renewal/PO workflows.
5. **Growth.** Sales & prospecting + Apollo; client success; signal alerts.
6. **Warehouse.** Loader → BigQuery; Dataform models; historical win-rate, pricing and contract economics.
