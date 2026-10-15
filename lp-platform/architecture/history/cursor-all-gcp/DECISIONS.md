# L&P platform — architecture decisions

Target architecture · 14 Oct 2026  
Diagram: [lp-platform-architecture.html](lp-platform-architecture.html)

**What decides this.** The Reference sheet (purpose, capabilities, features) and the owner's hosting decision. Google Cloud is the platform. The current Contract Intelligence Platform repository is a draft of behavior. It does not decide the host. Fly.io, Railway, Vercel, and Supabase are not this architecture.

## What it is

An internal operating platform for L&P Global Security: find the work, decide whether to bid, price it, write the proposal, submit that package outside the platform, then administer the contract and the recompete.

Staff use one web application. ChatGPT and Claude call the same services as the signed-in person. Buyers and competitors are records. They do not get logins.

## Where it runs

| Part | Google Cloud |
| --- | --- |
| Staff workspace, API, and agent tools | Cloud Run |
| Document parsing, OCR, schedule checks, exports, connectors | Cloud Run Jobs |
| Live business records | Cloud SQL for PostgreSQL |
| Original files | Cloud Storage |
| History, trends, and later accounting and workforce actuals | BigQuery, with Dataform for the metric definitions |
| Daily checks that only notify | Cloud Scheduler |
| Intake that must wait for a person | Cloud Workflows |
| Keys and logs | Secret Manager, Cloud Logging |

## Rules

1. **One place to edit the business.** Cloud SQL and Cloud Storage. BigQuery is a historical copy. Drive, Docs, and Sheets are sources or outputs.
2. **One set of business rules.** The screen and the agent call the same services.
3. **Agents act as the signed-in person.** No shared password. The platform does not call another model to answer a ChatGPT or Claude chat.
4. **Files become records only after a person checks them.** Extraction writes candidates. It cannot mark them verified.
5. **Authoritative outside systems stay outside.** QuickBooks, Belfry, and the existing CRM remain those systems. This platform stores the L&P view and the mismatches, and does not replace the books, the workforce tool, or the contact list.
6. **Core work does not need a model API key.** Search is full text. OCR of scans runs on the worker. An in-app model is optional.
7. **Five price columns, never blended.** Buyer requested, L&P cost, L&P proposed, buyer awarded, current contract. L&P cost is not a bill rate. A person approves the offer.
8. **Automation notifies.** It does not price, approve, or submit.
9. **Internal users only.** No buyer portal and no billing product.

## Still open

- Whether to reuse the current application or rebuild it on this layout.
- Connecting QuickBooks and Belfry.
- Win probability, until there is enough labeled history.
- One real bid carried from intake to a saved buyer-format package with a named list of what is still unresolved.
