# L&P Platform: Starter Stack

Researched 14 Oct 2026. Implements [DECISIONS.md](DECISIONS.md) (D1–D12). This is the "install first" subset of [FULL-STACK.md](FULL-STACK.md).

> ⚠ Step 8: research on 15 Oct 2026 found Supabase's OAuth 2.1 server not yet ready for ChatGPT/Claude. See FULL-STACK.md change C1 (Better Auth). This is pending the owner's decision.

## Verdict: minimal official scaffold, then add pieces by layer

Don't use an all-in-one template.

- **Official Vercel "Supabase Starter" (`with-supabase`).** Last updated 8 Oct 2026. It still ships Tailwind v3, `eslint-config-next@15.3.1`, and `next`/`@supabase/*` pinned to `"latest"`, which means unpinned. That is nearly identical to the old CIP repo's `apps/web/package.json`; CIP likely inherited those issues from it. The only useful part is the Supabase password-auth block, and that installs on its own (below).
- **SaaS boilerplates (MakerKit, Supastarter, ShipFast, Vercel SaaS Starter).** These are built to *sell* SaaS: Stripe billing, public sign-up, team invites, marketing pages. Design rule R9 (internal users only) makes most of that dead weight you'd have to rip out.
- **MCP templates.** Demo-sized (a dice-roll tool). The value is the `mcp-handler` package, which is installed directly.

## Install order

| # | Layer | Install | Source |
|---|---|---|---|
| 1 | Scaffold | `npx create-next-app@latest` → "recommended defaults" (TypeScript, ESLint, Tailwind, App Router, AGENTS.md). Check that `tailwindcss` is v4 and Node ≥ 20.9. | [create-next-app docs](https://nextjs.org/docs/app/api-reference/cli/create-next-app) |
| 2 | UI system | `npx shadcn@latest init` with **Radix** base (AI Elements needs Radix). Add components as features need them. | [shadcn](https://ui.shadcn.com) |
| 3 | Database client | `@supabase/supabase-js` + `@supabase/ssr` (cookie auth for App Router) | [Supabase SSR](https://supabase.com/docs/guides/auth/server-side/nextjs) |
| 4 | Auth UI | `npx shadcn@latest add https://supabase.com/ui/r/password-based-auth-nextjs.json` (+ dropzone block for uploads) | [Supabase UI Library](https://supabase.com/ui/docs/getting-started/introduction) |
| 5 | DB tooling | Supabase CLI: local stack, SQL migrations, `supabase gen types typescript` | Supabase CLI |
| 6 | Env wiring | Vercel Marketplace **Supabase integration** (syncs env vars); add Supabase Branching later for per-PR databases | [Vercel docs](https://vercel.com/docs/marketplace/supabase) |
| 7 | Workflows | `workflow` + wrap `next.config` with `withWorkflow()` | [useworkflow.dev](https://useworkflow.dev/docs/getting-started/next) |
| 8 | MCP server | `mcp-handler@2` + `@modelcontextprotocol/server@2` + `zod@4`; `withMcpAuth` + `protectedResourceHandler` pointing to Supabase OAuth 2.1 | [Vercel MCP docs](https://vercel.com/docs/mcp/deploy-mcp-servers-to-vercel), [Supabase MCP auth](https://supabase.com/docs/guides/auth/oauth-server/mcp-authentication) |
| 9 | GCP auth | `@vercel/oidc` + `google-auth-library` (Workload Identity Federation) | [Vercel OIDC → GCP](https://vercel.com/docs/oidc/gcp) |
| 10 | Worker | FastAPI service from the official Cloud Run quickstart, but with **your own Dockerfile** (Tesseract/OCRmyPDF are system packages buildpacks won't add). Batches: `gcloud run jobs deploy`. | [Cloud Run FastAPI](https://cloud.google.com/run/docs/quickstarts/build-and-deploy/deploy-python-fastapi-service), [Cloud Run jobs](https://cloud.google.com/run/docs/quickstarts/jobs/build-create-python) |

Add only when a feature needs it: TanStack Table/Query, React Hook Form, Tiptap, data grid, react-pdf, charts, Vercel AI SDK (`ai`).

## Rules

- **Pin versions.** Never `"latest"` in `package.json`; commit the lockfile.
- **One package manager** (pnpm) with workspaces: `apps/web`, `services/worker` (Python, uv), `packages/shared`.
- Each install is verified by its phase's "done when" check, not by the fact that it installed.
