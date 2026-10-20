# Belfry connector: findings

Investigated 20 Oct 2026 in Belfry's web app, signed in as Ashton (L&P's own tenant). Read-only: pages were viewed and one `SELECT` was run in Belfry's own report editor; nothing was saved, created or changed.

## Key finding: Belfry has a customer-facing SQL report editor

**Where:** Analytics → Dashboards → **Custom Reports** → Create report (`/analytics/custom-reports/new`).

**What it has** (observed):
- **Report editor tabs:** Catalog, Table Details, Filters, Chart, **SQL**.
- **SQL builder:** a code editor ("Cmd/Ctrl + Enter to run") with a **Run** button. Results appear in a paged grid with Columns, Filters and Density controls.
- **Test:** `SELECT COUNT(*) AS sites FROM customer` returned `284`. Table names in SQL are lowercase (`customer`); one column description references `shiftinstancedetails`, so multi-word tables may have no underscores. Confirm with the catalog.
- **Data catalog:** 128 tables in 7 groups, each with a plain-English description.
- **Table Details:** for each table, every column with its type, a description, allowed values for enums, and notes, e.g. "Filter active/in-progress shifts: `clock_in IS NOT NULL AND clock_out IS NULL`". Money is stored in cents (e.g. `billing_rate`, `post_pay_rate`).
- **Saved reports:** reports can be saved with a name and description and listed at `/analytics/custom-reports`. Filters turn hard-coded values into reusable inputs.
- **Belle:** Belfry's AI assistant ("Ask Belle to build one from the chat sidebar") builds dashboards from this same catalog.

**Why this matters:** this is a better connector than the REST endpoints. One "run SQL" call covers every table, and the catalog is already documented well enough for an AI agent to write correct queries. It uses the feature as Belfry designed it for customers. It is not the embedded Metabase dashboards (`/api/v1/metabase/iframe`), which we do not touch.

**Unknown:** the HTTP request behind **Run** (path, body shape, result format, row limits, timeouts) and behind the catalog. These could not be observed from Claude Code: listing the page's API calls is blocked by Claude Code's safety check while a person is signed in. Capture them with the gateway's own `login --capture` (see spec), which records method, path, query keys and body/response *shape* without values.

## Data catalog (128 tables)

**Accounting (7):** Customer Dimension Template Assignment · Dimension Instance · Dimension Key · Dimension Template · Dimension Template Instance · Dimension Value · Workplace Dimension Template Assignment

**Billing (22):** Aux Billing Item · Billing Result (billing line per shift instance) · Consolidated Activity Report · Contract (billing settings per site, OT rules) · Contract Billing Template · Contract Billing Template Holiday · Customer Service Rate (pay and bill rate per customer per service) · Custom Report (officer-filed field reports) · Firm Service · Firm Usage · Firm Usage Officer Detail · Holiday · Invoice · Invoice Line Item · Invoice Memo · Invoice Shift Instance · Parent Customer · Payment Request · Payment Transaction · Removed Invoice · Service Billing Settings · Shift Instance Billing Override

**HR (20):** Certification · Document Sign Instance · Employee Action · Employee Note (incl. disciplinary) · Everify Case · Issued Certification · Issued License (state guard licenses, expirations) · License · Onboarding Action · Onboarding Document · Onboarding Status · Onboarding Template · Tag · Tag Assignment · Time Off · Time Off Policy · Time Off Policy Mapping · Time Off Request · Workers Comp Code · WOTC Applicant

**Operations (30):** Break Attestation Setting · Checkpoint · Checkpoint Instance · Custom Report Template · Dispatch · Distribution Setting · Duty · Email Communication · Email Template · Incident · Industry · Issue (detected → resolved episodes) · Mass Announcement Recipient · Notice · Notice Ack · Notification · Notification Log · Patrol Site Configuration · Post Order · Post Order Instance · Report Metadata (index of every filed report) · Runsheet · Runsheet Task · Runsheet Template · Settings · Shift Attestation Settings · SMS Communication · Standard Workers Comp Code · Tour · Tour Instance

**Payroll (25):** Benefit · Break Premium · Contractor Deduction · Contractor Deduction Override · Contractor Payroll · Custom Deduction Type · Deduction · Employee Payroll · Employee Payroll Earning · Employee Payroll Reimbursement · Employee Payroll Tax · External Benefit Reference · Non Hourly Pay Item · Officer Payroll Artifacts Log · Pay Group · Pay Period · Payroll · Pay Schedule · Reimbursement Policy · Reimbursement Type · Supplemental Pay Benefit · Supplemental Pay Benefit Instance · Time Differential · Wage Result · Wage Rule

**Scheduling (16):** Activity (clock in/out, breaks, mobile events) · Availability Block · Break Allotment · Break Attestation · Break Attestation Instance · Break Instance · Break Premium Configuration · Call Off · Position · Shift (master schedule) · Shift Attestation · Shift Attestation Packet · Shift Attestation Packet Event · Shift Instance (each worked/scheduled shift: clock times, pay and bill rates in cents) · Shift Offer · Signature

**Workforce (8):** Contact · Customer (sites and patrols) · Mass Announcement · Officer · Officer Banned Site · Region · Site · Supervisor Association

## How custom reports map to the platform objectives

| Objective | Tables |
|---|---|
| Pricing engine, staffing actuals | shift instance, wage result, billing result, customer service rate, contract, wage rule, time differential |
| Contract economics, billing exceptions | invoice, invoice line item, invoice memo, payment transaction, shift instance billing override |
| Credentials and readiness | issued license, issued certification, license, certification |
| Client issues, reviews | incident, issue, report metadata, custom report, dispatch, call off |
| Turnover risk | officer, call off, employee note, shift offer |

## The REST endpoints (secondary)

[`../belfry-endpoints.md`](../belfry-endpoints.md) lists ~70 GET endpoints the web app uses. Keep them for the few cases SQL doesn't cover (e.g. the live dashboard); otherwise prefer SQL.

## State of the BI platform repo (from a read-only code review)

`AshtonMedina22/Business-Intelligence`, local `Documents/Projects/Business-Intelligence`, `main` at 4442cc9:

- **Belfry work is uncommitted and was changing during the review.**
  - Changed files: `src/gateway/core/catalog/belfry.ts`, untracked `belfry-endpoints.ts`, `web-adapter.ts`, `node/session/{capture,login-worker}.ts`, `hosted-mcp.ts`, `hosted-browser.ts`.
  - It adds an Auth0 `auth` block that reads the token from sealed localStorage, plus REST read operations with empty `output_schema`.
- **Login and capture:**
  - The login worker seals cookies and localStorage (RSA-OAEP + AES-GCM).
  - `login --capture` records request shapes without values. It is the tool for discovering the Run endpoint.
- **Not built yet:**
  - **Token refresh:** none exists. A 401 maps to `reauth_required`.
  - **Pagination:** only offset style is supported.
  - **MCP surface:** the hosted Vercel MCP exposes only `web_*` browser tools; HTTP replay (`gateway_call_operation`) is only in the Supabase edge `gateway-mcp`.
  - **Sync:** there is no Cloud Run sync job; Cloud Run hosts only the browser worker (`bi-browser-worker`).
- **Unknown:** whether Belfry's Auth0 SPA keeps tokens in localStorage (`@@auth0spajs@@`) or in memory. If in memory, the sealed session has no bearer and replay fails until the login worker captures the token another way.
