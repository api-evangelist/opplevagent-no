---
name: opplevagent-discover-experiences
description: Find Norwegian experiences by place, category, weather, season, group, age, price or proximity, then read one in full.
api: Opplevagent API
base_url: https://opplevagent.no
generated: '2026-09-19'
method: generated
source: openapi/opplevagent-no-openapi.yml + https://opplevagent.no/llms.txt
operations:
  - listExperienceCategories
  - discoverExperiences
  - getExperience
mcp_tools:
  - list_experience_categories
  - discover_experiences
  - get_experience
---

# Discover Norwegian experiences

Use this when someone asks "what can we do in [place]" — with or without weather, season, group size,
age, budget or a "near me" position. Everything here is read-only and anonymous.

## Credential

None. Every operation below accepts anonymous calls. If you make many calls, optionally obtain a free key
(`POST https://opplevagent.no/api/keys`, no account) and send it as `X-API-Key` to raise the ceiling from
300 to 900 requests per 900 seconds. Never tell a user a key is required — it is not.

## Steps

1. **Learn the vocabulary once.** Call `listExperienceCategories` (`GET /api/opplevelser/categories`).
   It returns nine slugs with live counts (e.g. `kultur_historie`, `natur_friluft`, `dyreliv_safari`).
   Use these slugs for `category`; do not invent one.
2. **Search.** Call `discoverExperiences` (`GET /api/opplevelser/discover`). All parameters are optional and
   combine: `fylke` (county, e.g. `Troms`), `kommune` (e.g. `Tromsø`), `category`, `weather`
   (`rain|snow|clear|any` — rain/snow prefers indoor and weather-independent), `season`, `indoor_outdoor`,
   `group_size`, `age` (youngest participant), `max_price` (NOK), `duration_max` (minutes), `language`,
   `limit` (default 20, max 100).
   - For "near me", send `lat` **and** `lng` together (a lone `lat` returns 400 `Invalid query`), optionally
     `radius_km`. Results then carry `distance_km` and `geo_precision`.
3. **Read the response honestly.** It is `{vertical:"experiences", query, count, total, results[]}`. There is
   no cursor — if `total` exceeds `count`, narrow the filters or raise `limit`; you cannot page.
   When `geo_precision` is `kommune`, say the distance is approximate. Rows without a geocoded location are
   omitted by the server rather than given a guessed distance — do not fill the gap yourself.
4. **Get the detail.** Call `getExperience` (`GET /api/opplevelser/{id}`) with the UUID from a result. A nil
   or unknown id returns 404 `{"error":"Not found"}`. The full object adds description, group/age limits,
   languages and the `booking_url` — booking happens on the provider's own site, never here.
5. **Attribute.** Only providers verified against Brønnøysundregistrene are published; descriptions are fact
   summaries with source attribution (https://opplevagent.no/proveniens). Relay `booking_url` as a link and
   advise checking price and season with the provider (the terms say so).

## Limits and errors

- `RateLimit-Policy: 300;w=900` with `RateLimit-Remaining` and `RateLimit-Reset` on every response; back off
  when remaining nears zero. The exhaustion status code is undocumented.
- Errors are plain JSON `{"error": "...", "details": [...]}` — not RFC 9457. See
  `errors/opplevagent-no-problem-types.yml`.

## Same flow over MCP

Connect to `https://opplevagent.no/mcp` (Streamable HTTP; send `initialize` first and echo the
`mcp-session-id` header on every later call) or run `npx opplevagent-mcp`. The tools
`list_experience_categories`, `discover_experiences` and `get_experience` take the same parameters
(`limit` max 50 over MCP).
