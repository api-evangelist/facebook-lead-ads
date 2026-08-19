---
name: Retrieve leads from a Facebook lead ad form
description: >-
  Poll a Facebook Lead Ads leadgen form for the leads it has captured, page through the
  cursor-based collection, and read each lead's answer payload — with the permission,
  rate-limit and error rules this API actually enforces.
api: openapi/facebook-lead-ads-leads-api-openapi.yml
operations:
  - listLeadGenForms
  - listLeadsForForm
  - listLeadsForAd
  - getLead
  - bulkDownloadLeads
generated: '2026-08-14'
method: generated
source: >-
  openapi/*.yml (operationIds verified against the specs) plus
  conventions/, errors/, rate-limits/ and scopes/ in this repo.
---

# Retrieve leads from a Facebook lead ad form

Use this when a business wants the leads a Facebook or Instagram lead ad has captured and
does not have a webhook receiver. If real-time delivery is available, prefer the webhook
skill — polling is the fallback, and it is the expensive one on this API.

## Before you call anything

- **Base URL** is `https://graph.facebook.com/v22.0` in this repo's contract. The current
  Graph API version is **v26.0**, and the live API returns
  `x-ad-api-version-warning: You are calling a deprecated version of the Ads API.` for
  v22.0. Prefer a current version path unless a caller has pinned one.
- **Credential** is a **Page access token**, not an app token. Obtain it through Facebook
  Login / Meta Business Login as a Page admin.
- **Permissions**: `leads_retrieval`, `pages_show_list`, `pages_manage_ads`. To read
  ad-level fields (`ad_id`, `campaign_id`) you also need `ads_management` and
  `pages_read_engagement`.
- Send the token as `Authorization: Bearer <token>`.
- Meta asks agents to self-identify. Append to your existing User-Agent:
  `<AgentName>/<Version> (<ModelName>) <HTTPClient>`. Keep the name stable across requests;
  do not randomize and do not impersonate another agent.

## Steps

1. **Find the forms on the Page.** Call `listLeadGenForms` on `/{page-id}/leadgen_forms`.
   Pass `fields` explicitly — the Graph API returns a minimal default field set and will
   silently omit properties you did not ask for rather than erroring.
2. **List the leads.** Call `listLeadsForForm` on `/{form-id}/leads`. To scope by campaign
   instead of by form, call `listLeadsForAd` on `/{ad-id}/leads`.
   - Narrow with `filtering` — a JSON-encoded filter array. The documented operators are
     `GREATER_THAN`, `LESS_THAN` and `GREATER_THAN_OR_EQUAL`, commonly on `time_created`.
     Always filter on a watermark you stored; do not re-read the whole form.
3. **Page through the collection.** The response is the `GraphCollection` envelope:
   `data[]` plus `paging`. Follow `paging.cursors.after` (or `paging.next`) until it is
   absent. Do not compute offsets — this API is cursor-paged.
4. **Read a single lead.** Call `getLead` on `/{lead-id}`. The `Lead` schema is
   `id`, `created_time`, `ad_id`, `form_id` and `field_data` — an array of
   `{name, values[]}` answer pairs. `custom_disclaimer_responses` is documented but not in
   the spec. Field names in `field_data` are defined by whoever built the form, so map them
   by name and never by position.
5. **Backfill in bulk instead of paging.** For a historical load, call `bulkDownloadLeads`
   on `/{form-id}/bulk_leads` with `from_date` and `to_date` rather than walking the cursor
   from the beginning. Meta also publishes a CSV export at
   `https://www.facebook.com/ads/lead_gen/export_csv/`, which takes the same date bounds as
   POSIX timestamps. Use bulk for the first sync, cursors for the steady state.

## Rate limits — read this before you write a polling loop

The bucket that governs this product is the **LeadGen business use case**, and its quota
**scales with the number of leads generated**:

```
Calls within 24 hours = 4800 * Leads Generated
```

A brand-new form with three leads has a budget measured in thousands of calls, not
millions. A naive one-minute poll will exhaust it.

- Parse the **`X-Business-Use-Case-Usage`** response header. Its *value* is a JSON document
  keyed by business ID; `call_count`, `total_cputime` and `total_time` are percentages and
  `estimated_time_to_regain_access` is minutes.
- There is **no `Retry-After`** and **no 429**. Throttling arrives as **HTTP 400** with
  `error.code` `80005` (LeadGen), `80001` (Pages), `4` or `17` (platform). A client that
  backs off only on 429 will hammer this API straight through the limit.
- On Ads Insights buckets, *your own 4xx errors reduce your quota* — sloppy error handling
  is a rate-limit problem here.

## Errors

Almost everything is HTTP 400; the cause is in the body. Read `error.code`, not the status.

| `error.code` | Meaning | Do |
|---|---|---|
| 100 (subcode 33) | Object missing, invisible to this token, or edge unsupported | Check the id and the granted permissions. Do not retry. |
| 102 / 190 | Token expired or revoked | Re-run the login flow, get a new Page token. |
| 10 / 200-299 | Permission denied | A required permission was never granted. Do not retry. |
| 4 / 17 / 32 / 613 | Platform throttle | Back off, inspect `X-App-Usage`. |
| 80005 / 80001 | Business-use-case throttle | Back off by `estimated_time_to_regain_access`. |
| 1 / 2 | Transient | Retry with backoff. |

Always log `error.fbtrace_id` — it is the identifier Meta support asks for.

## Do not

- **Do not retry writes blindly.** This API publishes no idempotency contract at all: no
  `Idempotency-Key`, no client request id, no replay window.
- Do not assume `field_data` keys are stable across forms.
- Do not treat an empty `data[]` as an error — a form with no submissions returns an empty
  collection.

## Related

- `asyncapi/facebook-lead-ads-webhooks.yml` — real-time delivery instead of polling
- `rate-limits/facebook-lead-ads-rate-limits.yml`
- `errors/facebook-lead-ads-problem-types.yml`
- `conventions/facebook-lead-ads-conventions.yml`
