---
name: Run an incident investigation and retrieve the result
description: >-
  Trigger an IncidentFox AI investigation with a named agent, poll it to
  completion, and read the root-cause result - the marquee flow of the
  IncidentFox REST API.
api: IncidentFox REST API
base_url: https://api.incidentfox.ai/api/v1
operations:
- run_agent            # POST /api/v1/agents/run
- get_investigation_status  # GET /api/v1/agents/status/{investigation_id}
- get_effective_config  # GET /api/v1/config/me/effective
source: https://docs.incidentfox.ai/api-reference/introduction
---

# Run an incident investigation and retrieve the result

Use the IncidentFox REST API to launch an AI SRE investigation and collect its
root-cause analysis. All calls are JSON over HTTPS against
`https://api.incidentfox.ai/api/v1` and require a Bearer token.

## Authentication

Send a team or admin token on every request:

```
Authorization: Bearer tokid.toksecret
```

Team tokens (`tokid.toksecret`) can trigger investigations and read/write the
team config; admin tokens (`admin.tokensecret`) add org-wide access. Tokens are
issued by an org admin in the Web UI. See `authentication/brownie-authentication.yml`.

## Steps

1. **(Optional) Read the team's effective config** — `GET /config/me/effective`
   to confirm which agents, MCP servers, and tools are enabled before running.

2. **Trigger the investigation** — `POST /agents/run` with a JSON body:
   - `agent_name` (required): one of `planner`, `investigation_agent`,
     `k8s_agent`, `aws_agent`, `coding_agent`.
   - `message` (required): the natural-language investigation request.
   - `context` (optional object) and `async` (optional boolean).
   Set `"async": true` to return immediately with an `investigation_id`.

   ```bash
   curl -X POST https://api.incidentfox.ai/api/v1/agents/run \
     -H "Authorization: Bearer $TEAM_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"agent_name":"planner","message":"Investigate high latency in payments","async":true}'
   ```

3. **Poll for completion** — `GET /agents/status/{investigation_id}`. The
   `status` field moves `running` -> `completed` (or `failed`). While running,
   `progress` reports `current_step` / `steps_completed` / `steps_total`. On
   `completed`, read `result.summary`, `result.root_cause`, and
   `result.recommendations`.

4. **(Alternative) Subscribe to the webhook** — instead of polling, configure
   `webhooks.investigation_complete` in team config to receive the result via
   an `investigation_complete` event (see `asyncapi/brownie-webhooks.yml`).

## Conventions and error handling

- Successful responses wrap payloads in `{ "data": ..., "meta": { request_id, timestamp } }`.
- Errors return `{ "error": { "code", "message", "request_id" } }`. Handle
  `401` (bad/expired token), `403` (insufficient permission/scope), `404`
  (unknown `investigation_id`), and `429` (rate limited). See
  `errors/brownie-problem-types.yml`.
- Respect rate limits: investigation triggers are capped at 10/minute, reads
  100/minute (`rate-limits/brownie-rate-limits.yml`). On `429`, back off using
  the rate-limit response headers.
- No idempotency-key is documented; avoid blind retries of `POST /agents/run`
  to prevent duplicate investigations - prefer polling an existing
  `investigation_id`.
