# L&P Platform: Full Stack (top-down)

Researched 14–15 Oct 2026 against official docs, GitHub releases, npm and PyPI. Implements [DECISIONS.md](DECISIONS.md) (v3: Google Cloud for compute and analytics, Vercel for the app, Supabase for data). [STARTER-STACK.md](STARTER-STACK.md) is the "install first" subset of this file.

**How to read it.** Each layer lists its components. For each component there is a recommended option, credible alternatives and when you'd switch, and the build phase. Nothing is installed until its phase. Versions are what was current on the research date; pin exact versions when you install. `⚠` marks a finding that changes or sharpens an earlier decision. `?` marks something that still needs confirming.

**Phases:** P1 foundation · P2 one complete bid · P3 agents · P4 revenue data · P5 growth · P6 warehouse.

---

## ⚠ Changes to v3 this research surfaced (need your OK)

| # | Change | Why |
|---|---|---|
| C1 | **Use Better Auth, not Supabase, as the OAuth server ChatGPT and Claude sign in through** (`@better-auth/mcp` + `@better-auth/cimd`, tables in your own Postgres). WorkOS AuthKit is the managed fallback. | Supabase's OAuth 2.1 server is still public beta. It has no CIMD support ([#2850](https://github.com/supabase/auth/issues/2850)), which the MCP 2026-07-28 spec prefers. It doesn't bind tokens to the resource audience, which ChatGPT checks ([#2610](https://github.com/supabase/auth/issues/2610)). Users also report consent-screen bugs that block ChatGPT ([#2820](https://github.com/supabase/auth/issues/2820)). |
| C2 | **Decide whether Better Auth also handles staff sign-in** (one auth system), or Supabase Auth keeps staff login and Better Auth only handles agents. | Two auth systems means a linked-login design. One system is simpler and more portable, but loses the Supabase Auth UI blocks. **Lean: Better Auth for both, with Google Workspace as the only login method.** |
| C3 | **Belfry gets a file-import adapter, not an API connector.** | Belfry publishes no API, webhooks or developer docs. Ask Belfry about API access and scheduled exports. |
| C4 | **The BigQuery copy is a custom Cloud Run Job, not Supabase Pipelines or Datastream.** | Pipelines is alpha and keeps current state only, with no history. Datastream doesn't support tables with RLS policies. |
| C5 | **Embeddings use Gemini (`gemini-embedding-2`) through Vertex AI at 768 dimensions**, not OpenAI. | Keeps proposal text inside Google Cloud with the rest of the heavy processing. Full-text search remains the no-key fallback. |
| C6 | **All document generation (DOCX, PDF, XLSX) runs on Cloud Run**, not Vercel. | LibreOffice and Chromium binaries and long renders don't suit Vercel functions. |

---

## Layer 1: Presentation (Next.js on Vercel)

| Component | Recommended | Alternatives → when | Phase | Notes |
|---|---|---|---|---|
| Framework | Next.js 16 App Router, TypeScript strict, React 19 | none | P1 | `create-next-app` recommended defaults include AGENTS.md |
| UI kit | shadcn/ui (Radix base), Tailwind v4, Lucide | none | P1 | Radix is needed for AI Elements |
| Record tables, compliance matrix | TanStack Table **v9** (9.3.1, MIT) | Pin v8.21.3 → if you want the shadcn Data Table recipe unchanged | P2 | v9 shipped Aug 2026. Whether the shadcn recipe has been updated for v9 is unconfirmed `?` |
| Pricing worksheet grid | **AG Grid Community** (36.2, MIT) | Handsontable (commercial) or AG Grid Enterprise → only if users insist on Excel paste/formulas, and budget a license. RevoGrid (MIT) → lighter option `?` | P2 | ⚠ Glide Data Grid hasn't released since Feb 2024. Pricing math lives in TS services, not the grid. |
| Proposal editor | **Tiptap v3** (MIT core) | **Plate** (MIT comments + suggestions) → if track changes is a must and you won't pay for Tiptap. Lexical, BlockNote → not recommended | P2 | Tiptap comments, track changes and DOCX import/export are paid ($49–1,199/mo) |
| PDF evidence viewer | react-pdf 11 (MIT, PDF.js 6) | pdfjs-dist viewer → for search and thumbnails. EmbedPDF → for built-in annotation | P1 | Load it client-only; build highlights on the text layer |
| Forms | React Hook Form 7.90 + Zod 4 resolver | TanStack Form → for deep dynamic field arrays (wage tables) | P1 | One Zod schema shared with the server action and the MCP tool |
| Charts | Recharts 3 via shadcn charts | ECharts 6 → for dense dashboards and heatmaps. visx → for bespoke charts | P2 | Pin the version the shadcn chart block installs |
| Client cache | **None at first** (RSC reads + server actions) | TanStack Query 5 → for grid autosave, job-status polling, optimistic boards | P2+ | TanStack's own guidance: add it only when needed |
| Drag and drop (pipeline boards) | @dnd-kit/core 6 + sortable | Pragmatic Drag and Drop (Atlassian) → actively maintained, lower level | P5 | dnd-kit's stable line hasn't released since Dec 2024 |

## Layer 2: API, identity and agent gateway (Vercel)

| Component | Recommended | Alternatives → when | Phase | Notes |
|---|---|---|---|---|
| App API | Server actions (UI mutations) + route handlers (webhooks, jobs, polling) | next-safe-action → typed middleware | P1 | Server actions are public POST endpoints, so re-check auth inside every one |
| Staff sign-in | Google Workspace only, enforced 3 ways: consent screen **Internal**, `hd` hint, server-side domain check | — | P1 | Provider depends on C2. Supabase Auth: before-user-created hook. Better Auth: domain check in its config |
| Agent OAuth server | ⚠ **Better Auth `mcp()` + `cimd()`** (C1) | WorkOS AuthKit → managed. Supabase OAuth → once #2850/#2610/#2820 are fixed | P3 | Needs: CIMD, PKCE S256, refresh rotation, resource-bound tokens. Redirects: `claude.ai/api/mcp/auth_callback`, `chatgpt.com/connector_platform_oauth_redirect`, Claude Code loopback |
| MCP server | `mcp-handler` 2.3 + `@modelcontextprotocol/server` 2 + Zod 4 | xmcp → if you want a framework | P3 | `withMcpAuth` verifies tokens (JWKS via `jose`, check iss/exp/aud). Add `readOnlyHint` on read tools |
| Permissions | App-level `can(actor, action, resource)` in services (primary) + RLS deny-by-default (backstop) | — | P1 | Tables: `roles`, `role_permissions`, `user_roles`. Same `actor` from the cookie (UI) or the token (MCP) |
| Audit log | App-owned append-only `audit_events` (actor, ui/mcp, client, tool, before/after) | pgaudit → database-level logging | P1 | Insert through a security-definer function; no UPDATE/DELETE. `supa_audit` is archived |
| MCP testing | MCP Inspector 2.11 + `@modelcontextprotocol/conformance` (pre-1.0) | — | P3 | Then test as real connectors in Claude and in ChatGPT developer mode |

## Layer 3: Domain services (TypeScript, in the Next.js app)

| Component | Recommended | Alternatives → when | Phase | Notes |
|---|---|---|---|---|
| Service pattern | `server-only` modules per domain (`domain/capture`, `proposal`, `pricing`, `intel`, `contracts`, `sales`, `clients`, `work`, `dataops`). Each takes Zod-parsed input + `actor`, checks permissions, runs SQL and returns a DTO | — | P1 | The Next.js recommended "Data Access Layer". UI, MCP and workflows are thin adapters over it |
| Database access | **Drizzle ORM 0.45** on postgres.js | **Kysely** → most "plain SQL", no schema DSL. Prisma → not recommended (release state unclear, pooling friction) | P1 | Drizzle 1.0 is still RC: pin 0.45, use the core builder + `sql`, upgrade once |
| Migrations | **Supabase CLI owns all migrations** (plain `.sql`) | — | P1 | Never run `drizzle-kit migrate/push` against Supabase. Run `drizzle-kit pull` after each migration for types |
| Connection from Vercel | Supavisor transaction mode, port 6543, `prepare:false`, `max:1` | — | P1 | Drizzle bypasses RLS, so services enforce permissions (above) |
| Supabase keys | New `sb_publishable_` / `sb_secret_` keys | — | P1 | ⚠ Legacy anon/service_role keys are deprecated by end of 2026 |
| Domain calculators | Pure TS: cost buildup, relief factor, bill rate, margin | — | P2 | Called by UI, MCP tools and workflows alike |

## Layer 4: AI add-ons (optional; each has a no-key fallback)

| Component | Recommended | Alternatives → when | Phase | Fallback |
|---|---|---|---|---|
| LLM SDK | Vercel AI SDK v7 (`generateText` + `Output.object`) | — | P2+ | Feature hides itself; agents reason via MCP |
| Model access | Vercel AI Gateway (GA, zero markup, fallbacks, budgets) | Vertex AI direct → keep all model calls in GCP | P2+ | — |
| Embeddings | ⚠ Gemini `gemini-embedding-2`, 768 dims, via Vertex (C5) | OpenAI text-embedding-3-small ($0.02/1M) → cheapest API. Self-host bge-m3 / nomic on Cloud Run → no API at all | P2 | Full-text only. Store `embedding_model` per row; models aren't interchangeable |
| Hybrid search | Postgres `tsvector` (GIN) + pgvector `vector` (HNSW) + `hybrid_search()` with reciprocal rank fusion | — | P2 | Plain SQL, portable to Cloud SQL. Supabase ships pgvector 0.8.2 |
| LLM extraction | **LangExtract** (Google, Apache-2.0) over Docling output: every value tied to a source offset | Gemini/OpenAI structured output with `source_quote` → simpler. Document AI custom extractor `?` | P2+ | Rule-based extraction. Unverified spans go to human review |
| Cloud OCR | Google Document AI (OCR ~$1.50/1k pages `?`) | Mistral OCR → strong layout `?` | P2+ | OCRmyPDF/Tesseract |

## Layer 5: Orchestration and scheduling

| Component | Recommended | Alternatives → when | Phase | Notes |
|---|---|---|---|---|
| Durable workflows | **Vercel Workflow** (`workflow` 5.2, Apache-2.0), with hooks for human waits | Trigger.dev → long compute without Cloud Run. Inngest → richer flow control. Cloud Workflows → GCP-internal chains only | P1 | $0.02/1K events, ~3 events per step. Limits: 10k steps/run, 800 s per step on Pro. Use child workflows past ~2k events |
| Dispatch to Cloud Run | Step gets a Vercel OIDC token → WIF → ID token for the service URL (`run.invoker`). Jobs: call the `:run` API, and the job POSTs back to resume the hook | — | P1 | The ID-token step is inferred, not in Vercel's docs `?`. Use idempotency keys, since steps retry |
| Schedules: app | Vercel Cron → starts workflows (opportunity sync, alerts) | — | P1 | Pro: per-minute schedules |
| Schedules: database | Supabase Cron (pg_cron) → in-database SQL (expiry flags, view refresh) | — | P1 | ≤ 8 concurrent jobs, ≤ 10 min each |
| Schedules: GCP | Cloud Scheduler → Cloud Run Jobs (loads, crawls, ML, Drive/Gmail watch renewals) | — | P2 | Each schedule has exactly one owner |
| Queues | Not needed at first (workflow steps fan out) | Cloud Tasks → rate-limited fan-out (e.g. QuickBooks 500/min). Supabase Queues (pgmq) → transactional with DB writes | P4 | Vercel Queues is still beta |

## Layer 6: Processing and integration (Python on Cloud Run)

| Component | Recommended | Alternatives → when | Phase | Notes |
|---|---|---|---|---|
| Service framework | FastAPI + Pydantic 2 + **uv** (`uv.lock`) on `python:3.12-slim-bookworm`; one image, two entrypoints (service + job) | Litestar → no strong reason | P1 | Limits: 32 GiB / 8 vCPU, 60 min per request, 168 h per job task, 32 MiB request body (pass Storage paths, not files) |
| PDF / layout extraction | **Docling** (MIT): layout, tables, page and bbox locators | pdfplumber (MIT) → fast path for born-digital PDFs. pypdf → forms and metadata | P1 | ⚠ Avoid PyMuPDF (AGPL) and Marker weights (revenue cap). Pin Docling; it releases often |
| OCR | **OCRmyPDF 17 + Tesseract 5.5** (searchable PDF/A, original kept) | PaddleOCR, docTR → poor scans. Document AI → optional cloud (Layer 4) | P1 | Ghostscript is AGPL as a separate binary; note it in the license register |
| Spreadsheets in | **openpyxl** (formulas, merged and hidden cells, A1 locators) | python-calamine → fast bulk reads and `.xls`. pandas/polars → analysis | P1 | openpyxl's last release was 2024 (stable) |
| DOCX in | Docling (structure) + python-docx (exact run/table locators) | docx2python → headers, footers, comments | P1 | — |
| DOCX out | **docxtpl** (Jinja in Word templates, LGPL as a library) + python-docx | `docx` npm → code-built docs. docxtemplater → paid modules | P2 | — |
| PDF out | **Gotenberg 8** as its own Cloud Run service (LibreOffice + Chromium) | WeasyPrint → HTML reports. Playwright print | P2 | Official `-cloudrun` image tags exist |
| XLSX out (buyer templates) | openpyxl writes values into the buyer's workbook, then a LibreOffice recalculation pass | — | P2 | ⚠ Avoid ExcelJS (stale) and SheetJS npm (stuck at 0.18.5) |
| Web acquisition | **Crawlee for Python + Playwright**, as Cloud Run Jobs | Scrapy → high-volume static pages. Crawl4AI → LLM-ready markdown | P1 | ⚠ Firecrawl self-host is AGPL. Respect robots.txt and terms; never automate logins or CAPTCHAs; prefer official feeds (SAM.gov API, ESBD) |
| Google Workspace | Service account + **domain-wide delegation** on a dedicated automation user and Shared Drive; per-user OAuth only for "as me" actions | — | P2 | Drive watch channels last ≤ 7 days, so a Scheduler job renews them. Docs API: 600 writes/min per project |
| Shift coverage solver | **OR-Tools CP-SAT** (Apache-2.0) | — | P2 | Hard constraints: licensing, hours caps, rest. Objective: overtime and continuity |
| Pwin model | scikit-learn + `CalibratedClassifierCV`, time-based splits, Brier score | — | P6 | Small data, so simple regularized models with intervals |

## Layer 6b: Connectors (business systems)

| Source | Approach | Phase | Notes |
|---|---|---|---|
| Framework | **Hand-written adapters**: source ID, sync log, mismatch queue. Thin TS or Python | P4 | dlt/Airbyte have no verified QuickBooks source worth the weight at this scale |
| QuickBooks Online | OAuth 2.0 (`intuit-oauth`), CDC endpoint + webhooks (IDs only) | P4 | 500 req/min per company. CDC looks back 30 days max, so a longer outage needs a full re-query. Builder tier of the Intuit App Partner Program is free (500K reads/month) |
| Belfry | ⚠ **File import** (CSV/XLSX via upload, Drive or Gmail attachment) | P4 | No public API. Ask Belfry about API access and scheduled exports `?` |
| Apollo.io | REST API (credits per enrichment; 200/min on Basic/Professional) | P5 | Which plans include API access needs confirming with Apollo `?` |
| Gmail | Pub/Sub push + `users.watch` (renew daily) + `history.list` | P4 | Internal OAuth consent or domain-wide delegation |

## Layer 7: Operational data (Supabase)

| Component | Recommended | Phase | Notes |
|---|---|---|---|
| Database | Supabase Postgres: pgvector, pg_cron, RLS, plain SQL | P1 | Portable to Cloud SQL (pgvector 0.8.5 and pg_cron are supported there) |
| Files | Supabase Storage: append-only, `org/doc/version/sha256` paths | P1 | ⚠ Storage objects are **not** in Supabase backups (see Ops) |
| History for analytics | Trigger-maintained history tables (`valid_from`/`valid_to`) for key entities | P2 | Gives the warehouse real history (SCD2) |
| Environments | 3 fixed projects: dev / staging / prod | P1 | Supabase Branching (beta; ~$0.013/h per branch, outside the spend cap) only when concurrent PRs need it |

## Layer 8: Analytics (Google Cloud)

| Component | Recommended | Alternatives → when | Phase | Notes |
|---|---|---|---|---|
| Load to BigQuery | ⚠ **Custom incremental loader as a Cloud Run Job** (`updated_at` watermark + append-only history tables), on Cloud Scheduler | dlt `sql_database` → if you want a library. Supabase Pipelines → revisit at GA | P6 | Datastream can't read RLS-protected tables. BigQuery federated queries support Cloud SQL only |
| Transform | **Dataform** (free, native to BigQuery) | dbt → snapshots, MetricFlow semantic layer (v2 license unclear `?`) | P6 | — |
| Metric definitions | One `metrics` spec in the repo that generates both Postgres views and Dataform SQLX, plus a nightly reconciliation assertion | — | P4 | Rule 8: metrics defined once |
| Reporting | **In-app charts via the API** | Data Studio (free) → owners' ad-hoc analysis. Metabase → self-serve exploration | P6 | — |

## Cross-cutting: delivery, operations, quality

| Component | Recommended | Alternatives → when | Phase | Notes |
|---|---|---|---|---|
| Repo layout | pnpm workspaces + Turborepo: `apps/web`, `services/worker` (uv), `packages/contracts`, `supabase/`, `infra/` | Plain pnpm `-r` → no caching needed. Nx → heavier | P1 | Turborepo calls Python via thin `package.json` scripts |
| Shared types TS ↔ Python | Pydantic → FastAPI OpenAPI → `@hey-api/openapi-ts` (client + Zod); CI fails if the generated code differs | JSON Schema-first codegen | P1 | hey-api is pre-1.0, so pin it exactly |
| CI/CD: web | Vercel Git integration (preview per PR) | — | P1 | — |
| CI/CD: worker | GitHub Actions → `google-github-actions/auth@v3` (WIF) → Artifact Registry → `deploy-cloudrun@v3` (services and jobs) | — | P1 | Pin actions by commit SHA |
| CI/CD: database | `supabase/setup-cli` → `db push` to staging, then prod behind a manual approval | — | P1 | — |
| Infrastructure as code | **OpenTofu** (MPL) or Terraform (BSL), Google provider v8, state in GCS. Scope: projects, IAM, WIF pools, Artifact Registry, secrets, budgets, buckets | Pulumi → TypeScript instead of HCL. gcloud scripts → week one only | P1 | It's your record of Google Cloud IAM while you learn it |
| Secrets | Vercel env vars (per environment) · Google Secret Manager · Supabase new keys | — | P1 | One GCP project per environment |
| Errors | **Sentry** (`@sentry/nextjs` + `sentry-sdk` for FastAPI) | — | P1 | Free plan: 1 user, 5k errors/mo. Team plan: $26/mo |
| Logs and traces | Vercel Observability · Cloud Logging (structured JSON) · `@vercel/otel` → Cloud Trace, passing `traceparent` to the worker | Vercel log drains (Pro, $0.50/GB) → if cross-system search gets painful | P1 | — |
| Testing | Vitest 5 · Playwright (against preview URLs) · pytest · pgTAP (`supabase test db`, focus on RLS) · MCP Inspector/conformance | schemathesis → API fuzzing | P1 | — |
| Code quality | ESLint + `eslint-config-next` (or Biome) · Ruff · TS strict · lefthook | — | P1 | Next 16 removed `next lint`. Check TypeScript 7 compatibility with Next before adopting it `?` |
| Email | **Resend** + React Email (free 3k/mo) | Gmail API with domain-wide delegation → internal-only, no new vendor. Postmark → deliverability | P2 | ⚠ SendGrid's free plan is gone |
| Backups | Supabase daily backups (Pro, 7 days) + nightly `pg_dump` and a **Storage → GCS copy** job | PITR add-on (~$100/mo) → when data loss tolerance is minutes | P1 | Do a quarterly restore drill into dev |
| Cost guardrails | GCP budgets at 50/90/100% (alerts only) · Cloud Run max instances · Vercel Spend Management · Supabase spend cap | — | P1 | The spend cap doesn't cover compute, branching or PITR |
| Agent and dev docs | Root `AGENTS.md` + `CLAUDE.md` pointer, per-app AGENTS.md, ADRs in `docs/adr/` | — | P1 | — |

---

## License watchlist

- **Avoid:**
  - PyMuPDF (AGPL)
  - Firecrawl self-host (AGPL)
  - Marker model weights (revenue cap)
  - Handsontable / HyperFormula (commercial or GPL)
  - BlockNote xl exporters (GPL)
  - SheetJS npm (stale)
- **OK as libraries:** docxtpl (LGPL), OCRmyPDF (MPL), ghostscript (AGPL, as a separate binary)
- **Paid add-ons to budget if needed:** Tiptap comments and track changes, AG Grid Enterprise, docxtemplater modules, Supabase PITR, Sentry Team

## Open questions to close

1. C1/C2: Better Auth for agent OAuth only, or for staff sign-in too.
2. Belfry: API access, or which scheduled exports are available.
3. Apollo: which plan includes API access.
4. Document AI and Mistral OCR pricing: confirm before budgeting.
5. Proposal editor: whether comments and track changes are a must. That decides Tiptap paid vs Plate.
6. Pricing grid: whether users need Excel copy/paste. That decides AG Grid Community vs a paid grid.
