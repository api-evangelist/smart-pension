---
name: Submit payroll contributions to a Smart Pension scheme
description: >-
  Run a UK pay period end-to-end against the Keystone API — refresh member state before payroll,
  post contributions for each employee, and reconcile the result. This is the flow every payroll
  integration must get right.
api: openapi/smart-pension-keystone-openapi.yml
provider: smart-pension
base_url: https://api.autoenrolment.co.uk
sandbox_url: https://api.sandbox.autoenrolment.co.uk
operations:
  - companiesEmployeesListWithHistoricalEmployeeData
  - companiesEmployeesOptStatesGet
  - contributionsCreate
  - contributionsList
  - contributionsUpdate2
  - summariesGet
  - summariesPayableGet
  - batch
generated: '2026-08-05'
method: generated
---

# Submit payroll contributions to a Smart Pension scheme

Keystone is the API behind the Smart Pension UK master trust. Note the vocabulary before you start:
an **employer is a `company`**, a **scheme member is an `employee`**, and the person administering
the employer is a **`customer`**. This mirrors the database, not the pensions domain.

Operation ids below come from `overlays/smart-pension-keystone-overlay.yaml` — the published spec
declares none of its own.

## Before you begin

- Get an OAuth 2.0 access token. For a payroll product acting for an employer, use the
  authorization-code grant with the `customer` scope. For a machine-to-machine integration, use
  client credentials and request `read:employees` and `read:contributions` — the docs state a
  `read` scope also permits writes. **Scopes must be enabled on your application by Smart
  (api@smartpension.co.uk) before they work.**
- Client-credentials tokens expire after **10 minutes**. Refresh mid-run on a long payroll.
- Send `Authorization: Bearer <access_token>`.
- Build against `https://api.sandbox.autoenrolment.co.uk` first; sandbox data is permanent.
- You need the `company_id`. Creating the company is optional — most payroll integrations attach to
  an employer that already exists.

## Step 1 — refresh member state before you run payroll

Never post contributions against a stale roster. Members opt out, opt back in, and change their
contribution percentage between pay runs, and the pension scheme is the system of record for that,
not your payroll.

```
GET /companies/{company_id}/employees?limit=100&offset=0
```
→ `companiesEmployeesListWithHistoricalEmployeeData`

Page with `limit` (max **100**, default 50) and `offset`. Follow the `links.next` relation in the
response envelope until it is absent — its absence, not a cursor, marks the last page. The docs
warn that offset paging is **not stable** if records are added while you page, so pull the roster
in one pass and do not interleave writes.

Narrow the payload with `fields[]` to keep the call fast:

```
GET /companies/{company_id}/employees?fields[]=id&fields[]=external_id&fields[]=email&limit=100
```

Match members to your payroll records on **`external_id`** — that is the payroll id you control.
Do not match on email.

For a specific member whose status you need to confirm:

```
GET /companies/{company_id}/employees/{id}/opt_state
```
→ `companiesEmployeesOptStatesGet`

Skip anyone who has opted out. Posting a contribution for an opted-out member is a business-logic
failure and returns **403**, not 422.

## Step 2 — post a contribution per employee

```
POST /companies/{company_id}/employees/{employee_id}/contributions
  ?contribution[gross_qualifying_earnings]=1000.00
  &contribution[period_type]=Monthly
  &contribution[starts_on]=2026-03-01
  &contribution[ends_on]=2026-03-31
```
→ `contributionsCreate`

Note the shape: Keystone takes write input as **bracketed query parameters**, not a JSON body. This
surprises most client generators.

Dates must be `YYYY-MM-DD`. Keystone's date parsing is lenient in a dangerous way — `17/06/2001` is
rejected to `null` rather than erroring, and bare integers like `101112` are silently accepted as
dates. Normalise every date yourself before sending.

A success returns **201** with the created contribution, including its `state`.

### There is no idempotency key

**This is the single most important thing to know about this API.** Keystone documents no
`Idempotency-Key` header, no replay window, and no safe-retry contract. A retried POST after a
timeout can create a **duplicate pension contribution**.

Protect yourself:

1. Before retrying, read back what exists:
   ```
   GET /companies/{company_id}/employees/{employee_id}/contributions
   ```
   → `contributionsList`
2. Match on your `starts_on` / `ends_on` period. If a contribution for that period is already
   present, do not re-post — correct it instead:
   ```
   PATCH /companies/{company_id}/employees/{employee_id}/contributions/{id}
     ?contribution[gross_qualifying_earnings]=1500.00
   ```
   → `contributionsUpdate2` (returns **204**, no body)
3. Keep your own ledger of (company, employee, period) → contribution id, keyed off `external_id`.

## Step 3 — batch instead of looping

Contribution creates are rate-limited to **250 requests per minute**, and employee list/create to
**80 per minute**. Everything else defaults to 500 per minute. A large employer will exceed these
if you loop.

```
POST /batch
```
→ `batch`

Up to **50 operations** per batch, and any Keystone endpoint may appear inside one. You get a single
consolidated response. Use it for anything over a few dozen members.

If your product already produces a **PAPDIS** CSV, prefer the light-integration path instead — see
the companion skill `smart-pension-import-papdis-file.md`.

## Step 4 — reconcile

```
GET /companies/{company_id}/contributions/summary
GET /companies/{company_id}/contributions/summary/payable
```
→ `summariesGet`, `summariesPayableGet`

Compare the payable total to what your payroll expects before telling the employer the run
succeeded.

## Error handling

Keystone does **not** use RFC 9457 problem+json. The envelope is `code` / `title` / `detail` /
`source`, and it arrives either as a bare object or as an array under `errors` — handle both.

| Status | Meaning | What to do |
|---|---|---|
| 401 | Token missing, expired, or scope not enabled | Refresh the token; a 10-minute client-credentials token expiring mid-run is the usual cause |
| 403 | Business rule refused it | Read `detail`. Common: caller role is `employee`, member already has a contribution for the period, member already re-enrolled |
| 404 | Wrong id, or the token cannot see that company | Check `company_id` before blaming `employee_id` |
| 409 | Not unique, or an async operation still running | Retry after a delay — **but read back first**, there is no idempotency |
| 422 | Validation failure | Read `source.pointer` for the offending attribute; several errors may return at once |
| 429 | Rate limit exceeded | Back off. No `Retry-After` is published, so pick your own base delay |
| 500 | Server error | The body is **HTML, not JSON**. Do not parse it. Capture the `x-request-id` response header and raise a ticket |

## Versioning

The current version is **v12**. If you pin, do it explicitly with `?version=12` or
`Accept: application/vnd.autoenrolment.v12+json`; an unpinned request always gets the latest
version, so a breaking change lands on you without warning. Responses are always shaped to the
version you asked for.

## References

- Conventions: `conventions/smart-pension-conventions.yml`
- Errors: `errors/smart-pension-problem-types.yml`
- Rate limits: `rate-limits/smart-pension-rate-limits.yml`
- Auth and scopes: `authentication/smart-pension-authentication.yml`, `scopes/smart-pension-scopes.yml`
