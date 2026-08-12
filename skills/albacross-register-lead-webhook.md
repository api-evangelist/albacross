---
name: Register a lead webhook and receive identified companies
description: >
  Subscribe to Albacross identified-company lead events by registering a webhook against a saved
  Segment, optionally enriched with matching contacts, and deregister it cleanly.
api: null
operations:
- 'GET /n8n/me'
- 'GET /n8n/segments'
- 'GET /n8n/buyer_personas'
- 'POST /n8n/hooks'
- 'PATCH /n8n/hooks/{id}'
- 'DELETE /n8n/hooks/{id}'
grounding: >
  Every path, method, request field and auth header below is read verbatim from Albacross's own
  MIT-licensed source at github.com/albacross/n8n-nodes-albacross (credentials/AlbacrossApi.
  credentials.ts and nodes/AlbacrossTrigger/AlbacrossTrigger.node.ts), and GET /n8n/me was
  confirmed live (401 unauthenticated) on 2026-08-12. Albacross publishes no OpenAPI for this API.
generated: '2026-08-12'
method: generated
---

# Register a lead webhook and receive identified companies

## What this is

Albacross's real-time push surface: when an identified company matches a saved Segment, Albacross
POSTs the lead to a URL you registered. This is the only way to consume Albacross data as events
rather than lookups.

Webhooks are an **Organisation-plan** feature marked "available upon request" on the pricing page.
Confirm entitlement before building.

## Verify the key first

```
GET https://api.albacross.com/n8n/me
Authorization: Api-Key <YOUR_API_KEY>
```

Albacross's own client uses this as its credential test. Generate the key in the Albacross app
under **Settings → Integrations → n8n**.

## Discover what you can filter on

```
GET https://api.albacross.com/n8n/segments          -> [{id, name}, ...]
GET https://api.albacross.com/n8n/buyer_personas    -> [{id, name}, ...]
```

Both return unpaginated arrays. A Segment scopes which companies trigger the hook; a Buyer Persona
is a saved contact-targeting profile you can use instead of manual job-title keywords.

## Register the hook

```
POST https://api.albacross.com/n8n/hooks
Authorization: Api-Key <YOUR_API_KEY>
Content-Type: application/json
```

```json
{
  "webhook_url": "https://your-app.example.com/hooks/albacross",
  "settings_page": "https://your-app.example.com/settings/albacross",
  "segment_id": 123,
  "conditions": { "updates": false },
  "name": "Production lead intake",
  "contacts": null
}
```

- `conditions.updates` — `false` sends each company **once** (new only); `true` fires on every
  qualifying visit, including returning companies. Default to `false` unless you genuinely want
  repeat events, or you will re-process the same account
- `contacts` — `null` for company data only. Supply an object to attach matching contacts:

```json
{
  "limit": 3,
  "with_emails": true,
  "with_phone_numbers": false,
  "must_have_contacts": true,
  "phone_number": false,
  "keywords": ["Head of Marketing", "CMO"],
  "export_buyer_persona": false,
  "country_filter_type": "based_on_lead_country",
  "countries": []
}
```

- Set `export_buyer_persona: true` and `buyer_persona_id` **instead of** `keywords` to select
  contacts from a saved persona
- `country_filter_type` is one of `all`, `based_on_lead_country`, `selected`; supply `countries`
  only for `selected`
- `must_have_contacts: true` suppresses the lead entirely when no contact is found — a real
  filter, not just enrichment. Contacts consume verified email/phone credits

Keep the returned `id`. You need it to patch or delete.

## Update and remove

```
PATCH  https://api.albacross.com/n8n/hooks/{id}     # send only the fields that changed
DELETE https://api.albacross.com/n8n/hooks/{id}     # stops delivery
```

**These writes carry no idempotency key.** Albacross publishes no `Idempotency-Key` header, and its
own client sends none, so a retried `POST /n8n/hooks` after a timeout will create a **duplicate
hook** and you will receive every lead twice. Before retrying a registration, confirm whether the
first attempt landed, and store the hook id transactionally.

Always `DELETE` on teardown. There is no expiry and no listing endpoint documented, so an orphaned
hook keeps delivering with no way to enumerate it.

## Receiving the event

Albacross POSTs JSON to `webhook_url` containing the company record, any requested contacts, and
event metadata. **No payload schema is published** — the help centre shows the output format only
as a screenshot, and the delivered shape is a per-account template you can ask support to change.
Parse defensively and pin the shape you actually observe.

## Verification — read this carefully

Albacross generates a token per webhook "to prove that traffic is coming from the Albacross
platform", and specifying it is **optional**. There is:

- no documented header name carrying the token
- no HMAC signature
- no timestamp and no replay protection

So a receiver **cannot cryptographically verify** an Albacross webhook. Treat the endpoint as
publicly reachable: use an unguessable URL path, enforce the shared token if you configure one,
rate-limit the endpoint, and never take a destructive action on webhook content alone — re-verify
against the Enrich API before writing to a system of record.

No retry, backoff or dead-letter behaviour is documented either. Assume at-most-once delivery,
respond 2xx fast, and process asynchronously.
