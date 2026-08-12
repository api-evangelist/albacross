---
name: Enrich a company from a website domain
description: >
  Resolve a website domain to the same Albacross firmographic company record the Reveal API
  returns, for list enrichment, form-fill enrichment and CRM hygiene.
api: null
operations:
- 'GET /enrich/companies/{domain}'
grounding: >
  Endpoint, auth header, response example, field table and status-code table read verbatim from
  Albacross's own developer documentation at https://docs.albacross.com/enrich, and confirmed by a
  live unauthenticated probe on 2026-08-12 (403). Albacross publishes NO OpenAPI for this API, so
  no operationId is cited and none was invented.
generated: '2026-08-12'
method: generated
---

# Enrich a company from a website domain

## What this is

The domain-keyed sibling of the Reveal API. Same `Company` object, different lookup key: give it
`example.com` instead of an IP address. Use it when you already know the company's website —
enriching a CRM list, a signup form domain, or an inbound lead.

Albacross documents this API on `docs.albacross.com/enrich` but publishes **no OpenAPI document**
for it. There is no machine-readable contract to generate a client from.

## Call it

```
GET https://api.albacross.com/enrich/companies/example.com
Authorization: Api-Key <YOUR_API_KEY>
```

Same custom auth scheme as Reveal — the `Api-Key ` prefix is required.

## Response

Identical shape to Reveal. See `data-model/albacross-data-model.yml` for the full field list:
`name`, `country`, `url`, `number`, `description`, `founded_year`, `address`, `linkedin_url`,
`employees`, `financial_report`, `nace_code`, `linkedin_industry_code`. Only `name` and `country`
are required.

## Handle the outcomes

| Status | Meaning | What to do |
|---|---|---|
| 200 | Company matched | Use the record |
| **204** | Valid domain, no company matched | Not an error. Record a miss; do not retry |
| 400 | Invalid domain format | Normalise the domain (strip scheme, path, `www.`) and retry once |
| 401 / 403 | Auth failure | Note the inconsistency: an anonymous call to Enrich returns **403 "No authorization header"** where the same anonymous call to Reveal returns **401 "Authentication required"**. Handle both as auth failures |
| 429 | Quota exceeded | Back off; no Retry-After is sent. Contact support@albacross.com |
| 500 | Server error | Retry with backoff, quoting `x-request-id` |

## Enriching a list

There is no batch endpoint. Enrich one domain per call, serially or with a small concurrency
bound, and hold a client-side budget: the quota is contract-negotiated, no number is published,
and no header tells you how much is left. A large list can exhaust a quota with no warning.

Normalise domains before calling — send `example.com`, not `https://www.example.com/pricing`.

## Errors

Proprietary envelope, free-text English, no stable codes:

```json
{"infos":[],"errors":["No authorization header"]}
```
