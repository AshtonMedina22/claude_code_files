# Belfry: internal API map

Mapped 20 Oct 2026 by watching which requests Belfry's own web app (`app.belfrysoftware.com`) makes while signed in as Ashton. Only pages were read; nothing was created, edited or deleted.

These endpoints are **not a public API**. Belfry can change them without notice. Ask Belfry for official API access or scheduled exports before relying on them for production syncs.

## How sign-in works

- **API host:** `https://api.belfrysoftware.com`, REST over JSON, versioned paths (`/api/v1` … `/api/v4`).
- **Auth:** Auth0 (`belfrysoftware.us.auth0.com`) single-page-app login with PKCE.
  - Token audience is `https://api.belfrysoftware.com`.
  - Belfry uses an Auth0 organization, so the login is tied to L&P's org.
  - Scopes include `offline_access`, which means Auth0 can issue refresh tokens. A saved session could stay valid for longer than one access token, but I haven't measured how long.
- **Calls:** requests carry a bearer access token, the standard Auth0 SPA pattern. Not inspected (token values were deliberately not read).
- **Gateway implication:** the gateway's Belfry connector can use direct API calls (the `web` kind) with the sealed Auth0 session instead of driving a browser. It needs to:
  - refresh tokens before they expire;
  - mark the connection `needs_login` when the refresh fails.

IDs are UUIDs (shown below as `{id}`).

## Endpoints by area (all GET, read-only)

### Customers, sites, contracts
| Endpoint | Page | Likely contents |
|---|---|---|
| `/api/v1/customers?only_archived=false` | Sites, Scheduling, Timekeeping | All customer sites |
| `/api/v2/parent_customers?only_archived=false` | Parents | Parent customers (companies owning several sites) |
| `/api/v1/customers/{id}` | Site details | One customer |
| `/api/v1/site/{id}` | Site details | Site record (address, settings) |
| `/api/v1/contract/{id}?is_parent=false` | Site details | Contract for the site |
| `/api/v1/customer/{id}/service_rate` | Site details | Bill rates by service |
| `/api/v1/firm/services` | Site details | Service catalogue |
| `/api/v2/shift?customer_id={id}&as_of_date=YYYY-MM-DD` | Site details | Shift / post definitions for the site |
| `/api/v1/customer/{id}/patrol-shift` | Site details | Patrol shifts |
| `/api/v1/site/{id}/site_officer_history` | Site details | Officers who worked the site |
| `/api/v1/supervisors?customer_region_id={id}` | Site details | Supervisors for the region |
| `/api/v1/customer/{id}/standard_workers_comp_code_mapping` | Site details | Workers' comp codes |
| `/api/v4/contracts/missing_billing_contacts` | Sites | Sites with no billing contact (38 today) |
| `/api/v1/region`, `/api/v1/industry` | several | Lookup lists |
| `/api/v1/runsheet_templates` | Site details | Run-sheet templates |

### People
| Endpoint | Likely contents |
|---|---|
| `/api/v1/officer?only_terminated=false&all_officers=false` | Active officers |
| `/api/v1/officer/certification-expiration-status` | License / certification expirations |
| `/api/v1/organization/roles` | User roles |
| `/api/v1/all_positions` | Positions |

### Scheduling and timekeeping
| Endpoint | Likely contents |
|---|---|
| `/api/v4/schedule?start_date=…&end_date=…` | Scheduled shifts for a date range |
| `/api/v3/schedule/customers?start_date=…&end_date=…` | Schedule grouped by customer |
| `/api/v3/timecards_shifts/period?start_date=…&end_date=…&target_date=…&include_break_premiums=false` | Timecards (actual hours) |
| `/api/v1/marketplace/pending-assignment-count?start_date=…&end_date=…` | Open shifts count |
| `/api/v1/time_off_requests_pending/count` | Pending time off |
| `/api/v1/pay_groups` | Pay groups |

### Billing and payroll
| Endpoint | Likely contents |
|---|---|
| `/api/v1/invoices/summary_metrics` | Billing totals |
| `/api/v3/accounts_receivable/invoices/existing` | Issued invoices |
| `/api/v3/accounts_receivable/invoices/nascent?min_date=…&max_date=…` | Invoices not yet issued |
| `/api/v2/payrolls?page=1&size=25` | Payroll runs (paged) |
| `/api/v2/payroll/credit_limit_usages?type=company` | Payroll credit usage |

### Analytics
| Endpoint | Likely contents |
|---|---|
| `/api/v4/dashboards?scope=owned&size=25` / `scope=shared` | Belfry's built-in dashboards |

## Not mapped yet
- **Bell Tower:** incident and daily activity reports (needed for client reviews).
- **Site tabs:** Schedule, Site & Billing Settings, Documents.
- **Chat, workflows, settings.**
- **Response field names:** I listed endpoints only; the next step is recording the JSON fields of each.

## What maps to the architecture
| Platform need | Belfry source |
|---|---|
| Contracts, sites, bill rates | customers, site, contract, service_rate |
| Staffing actuals for pricing | schedule, timecards_shifts |
| Credentials tracking | officer/certification-expiration-status |
| Contract economics, billing exceptions | invoices, missing_billing_contacts |
| Client issues | Bell Tower (not mapped yet) |
