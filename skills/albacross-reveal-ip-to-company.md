---
name: Reveal a company from an IPv4 address
description: >
  Turn a website visitor's IPv4 address into an Albacross firmographic company record — name,
  country, registration number, address, LinkedIn URL, employee band, revenue band and industry
  codes — and handle the no-match, bad-input and quota cases correctly.
api: openapi/albacross-reveal-openapi.yml
operations:
- 'GET /company/{ip_v4}'
grounding: >
  method+path, verified verbatim in openapi/albacross-reveal-openapi.yml. The Albacross-published
  spec declares NO operationId for this operation, so none is cited here; overlays/
  albacross-reveal-overlay.yaml proposes revealCompanyByIp as an API Evangelist addition only.
generated: '2026-08-12'
method: generated
---

# Reveal a company from an IPv4 address

## Before you start

You need an Albacross API key. It is **not self-serve**: API access is an Organisation-plan
feature marked "available upon request" on the pricing page, and keys are issued by an account
manager. Do not build a flow that assumes a user can sign up and get a key.

## Which host

Albacross has two live hosts for this same capability and has not said which is canonical:

- `https://reveal.api.albacross.com/company/{ip_v4}` — the host the published OpenAPI declares
- `https://api.albacross.com/reveal/company/{ip_address}` — the host the current developer portal documents

Prefer `api.albacross.com`, because that is what the current documentation shows and it is the
same host that serves Enrich and the n8n API. Both answered live on 2026-08-12.

## Authenticate

Send a static key in a custom auth-scheme header. This is **not** Bearer:

```
Authorization: Api-Key <YOUR_API_KEY>
```

The OpenAPI records only `apiKey in header, name: Authorization` and omits the `Api-Key ` prefix,
so a client generated straight from the spec will send the bare key and get a 401. Always add the
prefix.

## Call it

```
GET https://api.albacross.com/reveal/company/192.0.2.1
Authorization: Api-Key <YOUR_API_KEY>
```

Only dotted-decimal **IPv4** is accepted. IPv6 is rejected with a 400 and a dedicated
`ErrorMessageUnsupportedIPv6` message; Albacross lists IPv6 as roadmap only. Validate the address
client-side before spending a call.

## Read the response

`200` returns a `Company` object. Every field except `name` and `country` is optional — code
defensively:

- `url` is the company domain and is the only viable join key; there is **no Albacross-issued id**
- `employees` and `financial_report` are **bands** (`from`/`to`), never exact figures. Parse
  `from`/`to`; treat `range` as a display string
- `financial_report.currency` is per-record and not normalised, so revenue bands are not comparable
  across companies without an FX step
- `nace_code` and `linkedin_industry_code` are two independent taxonomies that disagree on the same
  company in Albacross's own examples. Pick one; do not merge them

## Handle the outcomes

| Status | Meaning | What to do |
|---|---|---|
| 200 | Company matched | Use the record |
| **204** | Valid IP, no company matched | **Not an error.** Record a negative lookup. Do not retry |
| 400 | Malformed IPv4, or an IPv6 address | Fix the input. Do not retry unchanged |
| 401 | Missing or invalid key | Check the `Api-Key ` prefix. Do not retry with the same credential |
| 403 | Key lacks permission for this endpoint | Escalate to the account manager |
| 429 | Request quota or identified quota exceeded | Back off exponentially — there is **no Retry-After** header. Contact support@albacross.com to raise the quota |
| 500 | Server error | Retry with backoff, quoting the request id |

The 204 case is the one agents get wrong most often. Treat it as a clean "no data", never as a
failure to retry.

## Errors do not match the spec

At runtime every error returns a proprietary envelope:

```json
{"infos":[],"errors":["Authentication required - please provide a valid API key"]}
```

The spec's `ErrorMessage` schema declares a single `message` property instead, so a generated
client will fail to deserialize real errors. Parse `errors[0]` as free-text English. There are no
stable error codes to branch on.

## Rate limits and tracing

No `X-RateLimit-*`, `RateLimit-*` or `Retry-After` header is returned on any response — you get no
advance signal and no backoff hint, only a bare 429 after the fact. Implement your own client-side
budget.

Capture the request id from the response and quote it to support@albacross.com:
`x-request-id` on `api.albacross.com`, `apigw-requestid` on `reveal.api.albacross.com`.

## Do not

- Do not batch — there is no bulk or collection endpoint, only one lookup per call
- Do not retry a 204 or a 400
- Do not cache indefinitely; firmographics change and Albacross publishes no cache guidance
