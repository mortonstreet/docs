---
title: "RevCenter API v1"
description: "Bearer-token API for web search, LinkedIn search, enrichment, usage, and MCP wrappers"
---

# RevCenter API v1

RevCenter API keys authenticate customer software, MCP servers, CLIs, and direct curl
calls without exposing Serper or Harvest credentials.

```http
Authorization: Bearer rvc_live_...
```

## Current capabilities

| Capability | Endpoint | Scope |
| --- | --- | --- |
| Web search | `POST /v1/data/search/web` | `data:search:web` |
| LinkedIn people search | `POST /v1/data/people/search` | `data:people:search` |
| LinkedIn profile enrichment | `GET /v1/data/people/:profileId` | `data:people:read` |
| LinkedIn company search | `POST /v1/data/companies/search` | `data:companies:search` |
| LinkedIn company enrichment | `GET /v1/data/companies/:companyId` | `data:companies:read` |
| Usage totals and cost events | `GET /v1/usage`, `GET /v1/usage/costs` | `usage:read` |

## Provision a key

```bash
pnpm --dir backend api-key:create \
  --workspace <workspace_id> \
  --name "Production API" \
  --scopes data:search:web,data:people:search,data:people:read,data:companies:search,data:companies:read,usage:read \
  --funding-mode contract_pool \
  --environment live \
  --rate-limit 120
```

The raw key is printed once. Store it in the customer's secret manager.

`--rate-limit` is requests per minute for that key. The default is `120/min`
when omitted.

## Rate limits and concurrent callers

Rate limits are per API key, not per human user. If ten workers share one key, they share
that key's minute bucket. If ten customers each have their own key, each customer gets
their own bucket.

At the default `120/min`, one key can average about `2 QPS`, with short bursts allowed
inside the minute. Higher-volume customers should receive a higher per-key limit, for
example `600/min` for about `10 QPS`.

The API accepts concurrent requests. Each request authenticates the key, checks scope,
resolves the workspace's RevCenter-provided or workspace-owned provider key, calls the
underlying data source, records usage, and returns structured JSON. Downstream protection
is separate: web search is QPS-limited and LinkedIn enrichment is concurrency-limited.

## Response shape

Successful responses use:

```json
{
  "data": {},
  "meta": {
    "request_id": "uuid",
    "usage": {
      "operation": "data.people.profile",
      "acquisition_units": 10000,
      "billable_units": 10000,
      "funding_source": "platform"
    }
  }
}
```

Errors use:

```json
{
  "error": {
    "code": "missing_api_key",
    "message": "Send Authorization: Bearer rvc_live_..."
  },
  "meta": {
    "request_id": "uuid"
  }
}
```

For idempotent billing on retries, send `Idempotency-Key` or `idempotencyKey` on
provider-spending calls.

## MCP and agent use

The API is designed to be easy to wrap as MCP tools: bearer-token auth, scoped keys,
JSON bodies, deterministic endpoints, structured JSON responses, request IDs, and usage
tracking. It is not itself an MCP server; an MCP server should wrap these endpoints as
tools such as `web_search`, `linkedin_people_search`, `linkedin_profile_enrich`, and
`linkedin_company_enrich`.

## Scopes

```text
data:search:web
data:people:search
data:people:read
data:companies:search
data:companies:read
usage:read
billing:read
searches:write
searches:read
feedback:write
*
```

## Web search

```bash
curl https://api.revcenter.ai/v1/data/search/web \
  -H "Authorization: Bearer $REVCENTER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "site:linkedin.com/in wealth advisor San Jose CFP impact investing",
    "num": 10,
    "surface": "search",
    "gl": "us",
    "hl": "en"
  }'
```

`surface` defaults to `search` and can be any supported Serper surface:
`search`, `images`, `news`, `maps`, `places`, `videos`, `shopping`, `scholar`,
`patents`, or `autocomplete`.

## People search

```bash
curl https://api.revcenter.ai/v1/data/people/search \
  -H "Authorization: Bearer $REVCENTER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "search": "wealth advisor",
    "location": "San Jose, CA",
    "page": 1
  }'
```

## People enrichment

```bash
curl "https://api.revcenter.ai/v1/data/people/dahn-lincoln" \
  -H "Authorization: Bearer $REVCENTER_API_KEY"
```

You can also pass a full LinkedIn URL:

```bash
curl "https://api.revcenter.ai/v1/data/people/profile?linkedin_url=https%3A%2F%2Fwww.linkedin.com%2Fin%2Fdahn-lincoln" \
  -H "Authorization: Bearer $REVCENTER_API_KEY"
```

The path segment is sent to Harvest as `publicIdentifier` unless it is a full URL
or numeric `profileId`.

## Company search

```bash
curl https://api.revcenter.ai/v1/data/companies/search \
  -H "Authorization: Bearer $REVCENTER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "search": "Merrill Lynch",
    "location": "California",
    "page": 1
  }'
```

## Company enrichment

```bash
curl "https://api.revcenter.ai/v1/data/companies/merrill-lynch" \
  -H "Authorization: Bearer $REVCENTER_API_KEY"
```

## Usage

```bash
curl https://api.revcenter.ai/v1/usage \
  -H "Authorization: Bearer $REVCENTER_API_KEY"
```

```bash
curl https://api.revcenter.ai/v1/usage/costs \
  -H "Authorization: Bearer $REVCENTER_API_KEY"
```

## Metering model

Public API responses expose blended RevCenter Acquisition Units, not underlying provider
prices.

```text
acquisition_units = wholesale_usd * 1,000,000
overage_billable_usd = wholesale_usd * 5
```

Workspace-owned provider keys are recorded as `workspace_byok`: RevCenter records the
activity but does not treat vendor spend as customer-billable acquisition overage.
