---
name: Read a member's pension value for a banking or benefits app
description: >-
  The read-only member journey — authenticate a scheme member with the employee scope and display
  their pension valuation, fund holdings and contribution history. This is the flow banking apps
  and HR/benefits platforms build.
api: openapi/smart-pension-keystone-openapi.yml
provider: smart-pension
base_url: https://api.autoenrolment.co.uk
sandbox_url: https://api.sandbox.autoenrolment.co.uk
operations:
  - listAllAccountsAssociatedWithTheAuthenticatedIdentity
  - employeesSession
  - valuationsList2
  - valuationsGet2
  - companiesEmployeesEmployeeFundHoldingsList
  - companiesEmployeesFundsSplitsList
  - contributionsList
  - companiesEmployeesOptStatesGet
  - employeesRetirementOptions
generated: '2026-08-05'
method: generated
---

# Read a member's pension value for a banking or benefits app

Smart's own Flows guide names this as a common integration: "It is common for banks to want to
display the pension valuation for an employee within their banking app. This is done by building
the auth flow and then a simple GET request to the /valuations endpoint for the associated employee
ID."

Operation ids come from `overlays/smart-pension-keystone-overlay.yaml`.

## Auth: this one needs the member, not your server

This is the flow where the **authorization-code grant is mandatory**. You are reading an
individual's pension savings, so you need a token issued for that resource owner — request the
`employee` scope. A client-credentials token has no user context and is the wrong tool here.

The documented authorization-code shape:

- `GET /oauth/authorize` on the identity host with `client_id`, `redirect_uri`, `response_type=code`,
  `scope=employee`, and `subdomain`.
- Authorization host: `https://id.autoenrolment.co.uk` (sandbox: `https://id.sandbox.autoenrolment.co.uk`).
- Token host: the same, at `/oauth/token`.
- Send `Authorization: Bearer <access_token>` on every call.

The `employee` scope is documented as: *manage the employee account — list employee contributions,
edit employee details and preferences.*

Register a partner app per environment at `https://partner.autoenrolment.co.uk/partners/sign-up`
(sandbox: `partner.sandbox.autoenrolment.co.uk`).

## Step 1 — resolve who the token is for

```
GET /accounts
```
→ `listAllAccountsAssociatedWithTheAuthenticatedIdentity`

Lists every account associated with the authenticated identity. A person can be a member of more
than one scheme, so do not assume one.

```
GET /employees/session
```
→ `employeesSession`

## Step 2 — read the valuation

The employee-scoped, company-free form is the one to prefer in a member-facing app:

```
GET /employees/valuations
```
→ `valuationsList2`

```
GET /employees/valuations/{id}
```
→ `valuationsGet2`

There is also a company-scoped form when you already hold both ids:

```
GET /companies/{company_id}/employees/{employee_id}/valuations
```
→ `valuationsList`

Related reads worth surfacing:

- `GET /employees/subaccount_groups_valuations` → `subaccountGroupsValuationsList2`
- `GET /companies/{company_id}/employees/{employee_id}/discounted_valuation` → `discountedValuationGet`

## Step 3 — fund holdings and splits

```
GET /companies/{company_id}/employees/{employee_id}/employee_fund_holdings
```
→ `companiesEmployeesEmployeeFundHoldingsList`

```
GET /companies/{company_id}/employees/{employee_id}/fund_splits
```
→ `companiesEmployeesFundsSplitsList`

```
GET /companies/{company_id}/employees/{employee_id}/employee_funds_valuations
```
→ `companiesEmployeesEmployeeFundsValuationsGet`

## Step 4 — contribution history and status

```
GET /companies/{company_id}/employees/{employee_id}/contributions
```
→ `contributionsList` — paginated, `limit` max **100**, follow `links.next`.

```
GET /companies/{company_id}/employees/{id}/opt_state
```
→ `companiesEmployeesOptStatesGet`

```
GET /employees/retirement_options
```
→ `employeesRetirementOptions`

## Make the calls cheap

Member-facing screens should not pull whole objects:

- `fields[]=name&fields[]=value` — return only what you render.
- `include[]=owner` — embed a related object instead of making a second round trip; the response
  keeps the hierarchy so you do not have to weave it together client-side.
- `filter[state]=paid` — filter server-side.
- `sort=created_at&direction=DESC` — newest first for a history list.

Default page size is 50, maximum 100.

## Handle this data carefully

This is UK pension savings data about an identified individual. It is personal financial data under
UK GDPR, and Smart's trust center records that customer PII is in scope of its ISO 27001:2022 and
SOC 2 Type 2 programs.

- Never log a valuation payload or an access token.
- Never cache a member's balance beyond the session without an explicit lawful basis.
- Scope tokens to `employee` only — do not reach for `customer` or `user` breadth you do not need.
- Access tokens are short-lived; treat expiry as normal, not as an error state.

## Error handling

| Status | Meaning | What to do |
|---|---|---|
| 401 | Token expired, or scope is not `employee` | Re-run the authorization-code flow. Do not silently fall back to a client-credentials token — it has no member context |
| 403 | Business rule refused it | Read `detail`; some operations are explicitly forbidden when the caller's role is `employee` |
| 404 | Wrong employee or company id | Resolve via `/accounts` rather than assuming |
| 422 | Bad parameter | Read `source.pointer` |
| 429 | Rate limited | Default 500 req/min; back off, no `Retry-After` published |
| 500 | Server error | Body is **HTML**, not JSON. Capture `x-request-id` |

## Verifying tokens

Keystone publishes its JWT signing keys as a JSON Web Key Set at
`https://api.autoenrolment.co.uk/.well-known/jwks` (sandbox:
`https://api.sandbox.autoenrolment.co.uk/.well-known/jwks`) — anonymous, RFC 7517. Note there is no
OIDC discovery document pointing at it, so hard-code the URL or configure it out of band. Captured
at `well-known/smart-pension-jwks.json`.

## References

- Conventions: `conventions/smart-pension-conventions.yml`
- Auth and scopes: `authentication/smart-pension-authentication.yml`, `scopes/smart-pension-scopes.yml`
- Errors: `errors/smart-pension-problem-types.yml`
