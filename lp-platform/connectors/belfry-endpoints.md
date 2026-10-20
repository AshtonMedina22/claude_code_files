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

### Customer page tabs
| Endpoint | Tab | Likely contents |
|---|---|---|
| `/api/v2/contact/{id}?is_parent=false` | Site & Billing Settings | Billing contacts |
| `/api/v1/firm/payment/settings` | Site & Billing Settings | Payment settings |
| `/api/v1/customer/{id}/files` | Documents | Files attached to the customer |
| `/api/v2/activities?size=25` | Bell Tower | Activity feed (also on the main Bell Tower page) |
| `/api/v4/schedule`, `/api/v3/schedule/customers` | Schedule | Same schedule endpoints as Scheduling |

### Bell Tower (field operations, incidents)
| Endpoint | Likely contents |
|---|---|
| `/api/v1/belltower/live` | Live dashboard (who is on post now) |
| `/api/v2/activities?size=25` | Officer activity log (paged) |
| `/api/v2/belltower-reports?page=0&pageSize=25` | Submitted reports (incidents, daily activity reports) |
| `/api/v2/belltower-report-types` | Report types |
| `/api/v1/report-templates`, `/api/v1/custom-report-templates` | Report templates |
| `/api/v1/report-template/default-incident-fields` | Standard incident fields |
| `/api/v4/issues/all?status=detected&unassigned=false` | Issues (missed check-ins, late arrivals, etc.) |
| `/api/v4/issue-configs` | Issue rules |
| `/api/v2/dispatch/paginated?size=25` | Dispatches (paged) |
| `/api/v1/dispatch/count_by_status` | Dispatch counts |

### More scheduling, people, timekeeping, billing
| Endpoint | Likely contents |
|---|---|
| `/api/v1/marketplace/shift-instances?start_date=…&end_date=…` | Open shifts posted for pickup |
| `/api/v1/officer/all_with_onboarding?only_terminated=false` | Officers with onboarding status |
| `/api/v1/time_off/range?start_date=…&end_date=…` | Time off in a range |
| `/api/v1/timecards/payroll_period/officers?start_date=…&end_date=…` | Timecards per officer for a pay period |
| `/api/v2/timecards_non_hourly/payroll_period?start_date=…&end_date=…` | Non-hourly pay items |
| `/api/v1/payroll/payschedule?start_date=…&end_date=…` | Pay schedule |
| `/api/v1/invoices/aging?bucket_count=4&bucket_days=30&age_invoice_by=invoice_date&target_date=…` | AR aging |

### Belfry Analytics pages
These pages embed Metabase dashboards through `/api/v1/metabase/iframe?dashboard_id=N` (hours 37, incidents 2, licenses 6, turnover risk 27, payroll changelog 4). They return a dashboard to display, not data rows, so they aren't useful for syncing. Rebuild these metrics from the endpoints above.

## Not mapped yet
- **Response field names.** Recording them means reading the page's signed-in API responses, which Claude Code's safety check blocks in a browser where a person is signed in. Capture them instead with the gateway's own sealed session (or official Belfry credentials) by calling each endpoint once from the connector code and saving the JSON keys.
- **Write endpoints.** Not needed; the platform reads only.
- **Chat, workflows, settings.**

## What maps to the architecture
| Platform need | Belfry source |
|---|---|
| Contracts, sites, bill rates | customers, site, contract, service_rate |
| Staffing actuals for pricing | schedule, timecards_shifts |
| Credentials tracking | officer/certification-expiration-status |
| Contract economics, billing exceptions | invoices, missing_billing_contacts |
| Client issues, incidents | belltower-reports, issues, activities, dispatch |
