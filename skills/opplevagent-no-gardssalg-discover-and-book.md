---
name: opplevagent-gardssalg-discover-and-book
description: Find a farm-sale drink producer and submit a pending visit request on a guest's behalf, without ever claiming a confirmed booking.
api: Opplevagent API
base_url: https://opplevagent.no
generated: '2026-09-19'
method: generated
source: openapi/opplevagent-no-openapi.yml + mcp/opplevagent-no-mcp-tools.json + https://opplevagent.no/llms.txt
operations:
  - discoverExperiences
mcp_tools:
  - discover_gardssalg
  - book_gardssalg
---

# Gårdssalg: find a producer and request a visit

Use this for breweries, cideries, wineries, distilleries, meaderies and farm cafés that sell from the
farm — a separate vertical from experiences. Discovery is available on REST and MCP; the booking request
exists only as an MCP tool (its REST twin `POST /api/opplevelser/book` is documented in llms.txt but is not
in the OpenAPI, so this skill grounds booking in the tool).

## Steps

1. **Find producers.** REST: `discoverExperiences` with `category=gardssalg_smaking` plus `fylke`, `kommune`,
   `producer_type` (`bryggeri|cideri|vingård|destilleri|gårdskafé|mjød|seltzeri`), `booking_live=true` (the
   literal `true` filters; omitted means no filter), or `q` to look up ONE named producer. MCP:
   `discover_gardssalg` with the same fields (`query` instead of `q`). The response has
   `vertical:"gardssalg"` and `GardssalgProducer` rows: `id` (this is the `provider_id` for booking), `navn`,
   `producer_type`, `profile_url` and `booking {live, mode, note}`.
2. **Check `booking.live` before offering to book.** A producer with `live:false` cannot be booked; the tool
   rejects it with a clear message. Do not promise otherwise.
3. **Collect the guest's own details.** `slot_at` (`YYYY-MM-DDTHH:MM`, Europe/Oslo), `party_size` (1-50),
   `guest_name`, `guest_email` — ask the guest; never invent them. Optional: `guest_phone`, `notes`,
   `experience_id`.
4. **Submit the request.** Call `book_gardssalg` with `provider_id` (from step 1) **or** `provider_query`
   (the name as the guest said it), and pass `requested_weekday` (e.g. `fredag`) when the guest named a day.
   Handle the documented non-error outcomes inside the result: `provider_ambiguous` (present `candidates[]`),
   `provider_not_found`, `weekday_mismatch:true` (offer `suggestions[]`), `outside_hours:true` (retry with
   `confirm_outside_hours:true` only if the guest insists). None of these create a request.
5. **Report it as PENDING.** On success read back `provider.navn` and `slot_at_local` to the guest and say the
   producer will confirm, propose another time or decline by e-mail. The tool never creates a confirmed
   booking, no AI agent can confirm one, and there is no payment — pickup/visit, pay on arrival.

## Guardrails

- `book_gardssalg` declares `idempotentHint:false` and there is no replay key: do not retry a call that
  returned success, or a second pending request may be created.
- There is no cancel operation for the guest or the agent; the guest's status link is read-only. Say so
  before submitting.
- Anonymous ceiling on `/mcp` is 200 requests per 900 s (`RateLimit-*` headers); 600 with the optional key.
