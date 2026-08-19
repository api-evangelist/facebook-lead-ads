---
name: Create and inspect lead generation forms
description: >-
  List the lead generation forms on a Facebook Page, create a new one, and read a single
  form back — with the at-most-once write discipline this API forces, because it publishes
  no idempotency contract.
api: openapi/facebook-lead-ads-leadgen-forms-api-openapi.yml
operations:
  - listLeadGenForms
  - createLeadGenForm
  - getLeadGenForm
generated: '2026-08-14'
method: generated
source: >-
  openapi/*.yml (operationIds verified against the specs) plus conventions/ and errors/
  in this repo.
---

# Create and inspect lead generation forms

A lead generation form is the object a lead ad points at. Everything else in this API hangs
off its id.

## Prerequisites

- **Page access token** with `pages_manage_ads`, `pages_show_list` and `leads_retrieval`.
- Page Admin access, or flexible permissions, on the Page that will own the form.
- Base URL `https://graph.facebook.com/v22.0` per this repo's contract; v26.0 is current.

## Steps

1. **List what already exists.** Call `listLeadGenForms` on `/{page-id}/leadgen_forms`,
   passing `fields` explicitly. **Do this before every create** — see the warning below.
2. **Create the form.** Call `createLeadGenForm` on `/{page-id}/leadgen_forms` with a JSON
   body:
   - `name` — the form's label
   - `questions` — the array of fields shown to the user
   - `privacy_policy` — required; lead ads will not run without one
   - `follow_up_action_url` — where the user is sent after submitting
   - `locale`

   The response is `{ "id": "<form-id>" }` and nothing else.
3. **Read it back.** Call `getLeadGenForm` on `/{form-id}` with `fields` to confirm the
   form materialised as intended.

## Writes are at-most-once — the important warning

This API has **no idempotency key, no client request id, and no replay window**. A retried
`createLeadGenForm` creates a **second form**. Meta's documented retry guidance (retry on
error codes 1, 2, 4, 17, 368) is about transient failures and carries no de-duplication
guarantee.

The safe pattern:

1. Call `listLeadGenForms` and check whether a form with your intended `name` already
   exists.
2. Issue `createLeadGenForm` **once**.
3. If the call fails ambiguously — timeout, connection reset, 5xx — **do not retry**. Call
   `listLeadGenForms` again and look for the form before deciding anything.

## Contract gaps to expect

`getLeadGenForm` declares a `200` with a description and **no response schema**, and
`createLeadGenForm`'s `questions` array is typed as bare objects. The contract will not tell
you the form's real shape — read the object reference, or read a form back and inspect it.

## Errors

Almost every failure is HTTP 400 with the cause in `error.code`:

- `100` (subcode `33`) — the page-id or form-id is wrong, or invisible to this token
- `10`, `200`-`299` — a required permission was never granted
- `102`, `190` — the token expired; re-authenticate
- `80004` — Ads Management business-use-case throttle
  (`300 + 40 * active ads` calls/hour at Standard Access)

Log `error.fbtrace_id` on every failure.

## Related

- `data-model/facebook-lead-ads-data-model.yml` — the Page → Form → Lead graph
- `conventions/facebook-lead-ads-conventions.yml` — the idempotency finding in full
- `plans/facebook-lead-ads-plans-pricing.yml` — Standard vs Advanced Access
