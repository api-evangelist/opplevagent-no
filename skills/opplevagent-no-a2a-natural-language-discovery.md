---
name: opplevagent-a2a-natural-language-discovery
description: Ask Opplevagent in plain Norwegian or English over the A2A JSON-RPC endpoint and read the Task artifacts back.
api: Opplevagent API
base_url: https://opplevagent.no
generated: '2026-09-19'
method: generated
source: openapi/opplevagent-no-openapi.yml + a2a/opplevagent-no-agent-card.json + https://opplevagent.no/llms.txt
operations:
  - getExperiencesAgentCardWellKnown
  - experiencesA2AJsonRpc
a2a_skills:
  - opplevelser_discover
  - opplevelser_info
  - opplevelser_categories
---

# Natural-language discovery over A2A

Use this when you are an A2A client, or when the user's request is a sentence rather than a filter set
("hva kan vi finne på i Oslo når det regner", "things to do in Trondheim under 500 kr").

## Steps

1. **Read the card.** `getExperiencesAgentCardWellKnown` (`GET /.well-known/agent-card.json`). It declares
   `url: https://opplevagent.no/a2a`, `preferredTransport: JSONRPC`, `protocolVersion: 1.0.0`, three skills
   (`opplevelser_discover`, `opplevelser_info`, `opplevelser_categories`) and `authentication.schemes: [none]`.
   `GET /a2a` returns the same card and doubles as a health check.
2. **Send a message.** `experiencesA2AJsonRpc` (`POST /a2a`) with a JSON-RPC 2.0 body:
   `{"jsonrpc":"2.0","method":"message/send","params":{"message":{"text":"hvalsafari i Tromsø"}},"id":"1"}`.
   `tasks/send` is accepted as a pre-0.3 alias; prefer `message/send`. Structured filters may be sent as
   `application/json` per the card's `defaultInputModes`.
3. **Read the Task.** A live call returns `result.taskId`, `result.status.state` (`completed`) and
   `result.artifacts[]`: a text part (`"Fant 20 opplevelse(r). / Found 20 experience(s)."`) and a data part
   whose `data.experiences[]` rows carry `id`, `provider_id`, `provider_match_status`, `title`, `title_no`,
   `slug`, `category`, county/municipality, price and booking fields. Streaming and push notifications are
   `false` in the card — poll nothing, the response is complete.
4. **Follow up by id.** For a single experience, use the `opplevelser_info` skill (JSON `{ "id": "<uuid>" }`)
   or drop to REST `getExperience`.

## Rules

- No credential. The optional `X-API-Key` (card `securitySchemes.consumerApiKey`) only raises the ceiling
  from 200 to 600 requests per 900 s on `/a2a`.
- The card's JWS signature (EdDSA, kid `lokal-a2a-2026`) has no published key location; treat TLS as the
  trust anchor.
- The A2A surface exposes discovery only — not the gårdssalg vertical and not booking. For those, use MCP.
