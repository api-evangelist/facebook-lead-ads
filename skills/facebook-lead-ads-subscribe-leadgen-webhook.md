---
name: Subscribe to real-time leadgen webhooks
description: >-
  Wire an app and a Facebook Page to receive a leadgen webhook the moment someone submits a
  lead form — the verification handshake, the signature check, the at-least-once delivery
  rules, and the follow-up read that actually returns the lead.
api: openapi/facebook-lead-ads-subscriptions-api-openapi.yml
operations:
  - subscribeAppWebhook
  - pageSubscribedApps
  - getLead
generated: '2026-08-14'
method: generated
source: >-
  openapi/*.yml (operationIds verified against the specs) plus
  asyncapi/facebook-lead-ads-webhooks.yml and conventions/ in this repo.
---

# Subscribe to real-time leadgen webhooks

Webhooks are the reason Facebook Lead Ads exists as an API rather than a CSV export: a lead
is worth the most in the first minutes after it is submitted. This is the delivery path to
build first; polling (`listLeadsForForm`) is the fallback.

Subscription is **two steps, not one**. The app subscribes to the *object and field*, then
each *Page* subscribes the app. Miss the second and you will register successfully and
receive nothing.

## Prerequisites

- A Meta app with a **Verify Token** set in the App Dashboard.
- An HTTPS callback endpoint that is reachable before you subscribe — Meta calls it during
  registration.
- Permissions: `pages_manage_metadata`, `pages_show_list`, `leads_retrieval`.
- The app secret, for signature verification.

## Steps

1. **Stand up the verification handshake.** Meta sends a `GET` to your callback with three
   query parameters:
   - `hub.mode` — always the literal `subscribe`
   - `hub.verify_token` — the string from your App Dashboard
   - `hub.challenge` — an integer

   Compare `hub.verify_token` against your configured value. If it matches, respond by
   echoing `hub.challenge` back in the body. If it does not, reject. Never echo the
   challenge without checking the token.

2. **Subscribe the app.** Call `subscribeAppWebhook` on `/{app-id}/subscriptions` with
   `object=page`, your `callback_url`, `verify_token`, and `fields` containing `leadgen`.
   The body is `application/x-www-form-urlencoded`.

3. **Subscribe the Page.** Call `pageSubscribedApps` on `/{page-id}/subscribed_apps` with
   `subscribed_fields` including `leadgen`, using the **Page access token** for that Page.
   Repeat for every Page whose forms you want to receive.

4. **Verify every incoming payload.** Each notification is a `POST` carrying an
   `X-Hub-Signature-256` header of the form `sha256=<hex>`. Compute HMAC-SHA256 over the
   **raw request body** using your app secret and compare in constant time.
   - Compute the digest over the bytes you received. Parsing the JSON and re-serialising it
     changes the bytes and breaks verification — this is the single most common integration
     failure on this surface.

5. **Read the lead.** The notification carries IDs, not answers. Its shape is
   `object`, then `entry[]` with `id` (the Page), `time`, and `changes[]` where
   `changes[].field == "leadgen"` and `changes[].value` holds `leadgen_id`, `page_id`,
   `form_id`, `adgroup_id`, `ad_id` and `created_time`.
   Call `getLead` on `/{lead-id}` with the `leadgen_id` to fetch `field_data`.

6. **Answer `200 OK` fast.** Acknowledge before you do the follow-up read. Meta retries
   immediately, then with decreasing frequency over the next **36 hours**, and drops the
   notification after that.

## De-duplicate — this is not optional

Delivery is **at-least-once**, ordering is not guaranteed, and this API publishes **no
idempotency contract of any kind**. Keep a set of seen `leadgen_id` values and drop
repeats. If you do not, a retry storm becomes duplicate rows in the customer's CRM.

## Rate limits

The follow-up `getLead` calls count against the **LeadGen** business-use-case bucket:
`4800 * leads generated` per 24 hours, surfaced on `X-Business-Use-Case-Usage`, throttled
as **HTTP 400 with `error.code` 80005** — not 429, and with no `Retry-After`.

## Testing

Use the **Lead Ads Testing Tool** (`https://developers.facebook.com/tools/lead-ads-testing`)
to create and delete test leads against a real form; it fires a real notification at your
callback. It requires a Meta Business login. In Development Mode, app users with a role can
submit and read leads without a live campaign.

## Related

- `asyncapi/facebook-lead-ads-webhooks.yml` — the full channel and payload catalog
- `sandbox/facebook-lead-ads-sandbox.yml` — the testing tools and their login gates
- `errors/facebook-lead-ads-problem-types.yml`
