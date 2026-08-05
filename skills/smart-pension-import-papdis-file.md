---
name: Import a PAPDIS contribution file into Smart Pension
description: >-
  The "light integration" path — POST a PAPDIS CSV to the imports endpoint instead of writing each
  contribution individually, then poll the import for per-row errors and surface them for
  correction. Same journey the employer portal's file upload performs.
api: openapi/smart-pension-keystone-openapi.yml
provider: smart-pension
base_url: https://api.autoenrolment.co.uk
sandbox_url: https://api.sandbox.autoenrolment.co.uk
operations:
  - importCreate
  - importList
  - importGet
  - importImportResultsList
generated: '2026-08-05'
method: generated
---

# Import a PAPDIS contribution file into Smart Pension

PAPDIS (Payroll and Pension Data Interface Standard) is the UK standard for handing payroll data to
a pension provider. If your payroll product already emits a PAPDIS CSV, this is the lowest-effort
integration Smart documents: you POST the file you already generate, and Keystone performs exactly
the journey an employer performs by hand in the portal.

Operation ids come from `overlays/smart-pension-keystone-overlay.yaml`.

## When to use this instead of per-employee writes

Use the import path when:

- You already produce a PAPDIS file (no new data mapping needed).
- The employer's scheme already exists — you only need the `company_id`.
- The pay run covers enough members that per-employee POSTs would hit the 250/minute contribution
  limit or the 80/minute employee limit.

Use `smart-pension-submit-payroll-contributions.md` instead when you need per-member control,
mid-run correction, or you are also creating members.

Either way you should still read member state back before the run — the import does not tell you in
advance who has opted out.

## Step 1 — upload the file

```
POST /companies/{company_id}/imports
```
→ `importCreate`

You need an OAuth token with the `customer` scope (authorization code), or client credentials with
the relevant `read:` scopes enabled on your application by Smart. Remember client-credentials
tokens live only **10 minutes**.

Send against `https://api.sandbox.autoenrolment.co.uk` first. Sandbox data persists, so you can
build a realistic test employer once and reuse it.

## Step 2 — poll the import

Keystone has **no webhooks and no event stream**. There is nothing to subscribe to; polling is the
documented and only pattern.

```
GET /companies/{company_id}/imports/{id}
```
→ `importGet`

```
GET /companies/{company_id}/imports
```
→ `importList` — paginated with `limit` (max 100) / `offset`; follow `links.next`.

Poll with backoff. The default rate limit is 500 requests per minute, so a sane poll interval is
never the constraint — the constraint is being polite.

## Step 3 — read the per-row results and surface them

```
GET /companies/{company_id}/imports/{import_id}/import_results
```
→ `importImportResultsList`

This is the point of the whole flow. Keystone returns the per-row error messages the portal would
have shown the employer. **Render them in your own UI** so the user corrects the file in your
product and re-submits, rather than being sent to the Smart portal. Smart's own guidance calls this
out as the expected behaviour of a light integration.

The error envelope is `code` / `title` / `detail` / `source`; the `title` is written to be shown
in-line with the action and the `detail` beneath it or as a tooltip. Use them as authored — they are
meant for end users.

## Step 4 — re-submit corrections

Correct the file and POST it again with `importCreate`.

**Watch for duplicates.** Keystone publishes no idempotency key. If an upload times out you cannot
tell from the API whether it landed. Before re-uploading:

1. `importList` and check whether an import for that period already exists.
2. If it does, read its `import_results` rather than uploading again.

## The other import surface: SSIF

Alongside PAPDIS, Keystone exposes an SSIF import family with header mapping — useful when your file
is not PAPDIS-shaped:

- `sSIFImportsCreate` — `POST /companies/{company_id}/ssif_imports`
- `sSIFImportsFileHeaders` — `GET /companies/{company_id}/ssif_imports/{ssif_import_id}/file_headers`
- `sSIFImportsAssignSSIFHeadersMapping` — `POST /companies/{company_id}/ssif_imports/{ssif_import_id}/assign_ssif_headers_mapping`
- `sSIFImportsSSIFImportResultsList` — `GET /companies/{company_id}/ssif_imports/{ssif_import_id}/ssif_import_results`
- `sSIFImportsSSIFImportMissingEmployeesList` — `GET /companies/{company_id}/ssif_imports/{ssif_import_id}/ssif_import_missing_employees`
- `sSIFImportsTemplate` — `GET /companies/{company_id}/ssif_imports/template`

The SSIF flow is: upload → read the file's headers → map them to Keystone fields → read results,
including which employees in the file could not be matched.

## Error handling

| Status | Meaning | What to do |
|---|---|---|
| 401 | Token expired or scope not enabled | Refresh; confirm Smart enabled the scope on your app |
| 403 | Business rule refused the import | Read `detail` |
| 404 | Unknown `company_id` or import id | Check the company the token can see |
| 413 | Payload too large | Split the file |
| 422 | The file or a parameter failed validation | Read `source.pointer`; multiple errors may return at once |
| 429 | Rate limited | Back off; no `Retry-After` is published |
| 500 | Server error | Body is **HTML**, not JSON. Capture `x-request-id` and raise a ticket |

## References

- PAPDIS help: https://www.smartpension.co.uk/help-centre-articles/about-papdis-files
- Fixing PAPDIS errors: https://www.smartpension.co.uk/help-centre-articles/fixing-errors-with-your-papdis-upload
- Conventions: `conventions/smart-pension-conventions.yml`
- Errors: `errors/smart-pension-problem-types.yml`
