---
name: Read and update IncidentFox team configuration
description: >-
  Read a team's fully merged effective configuration and apply partial,
  deep-merged overrides (enabled agents, MCP servers, tools, feature flags,
  webhooks) via the IncidentFox REST API.
api: IncidentFox REST API
base_url: https://api.incidentfox.ai/api/v1
operations:
- get_effective_config  # GET /api/v1/config/me/effective
- update_config         # PUT /api/v1/config/me
source: https://docs.incidentfox.ai/api-reference/config/update
---

# Read and update IncidentFox team configuration

IncidentFox configuration is hierarchical (organization -> group -> team).
This skill reads the merged result and applies team-level overrides.

## Authentication

`Authorization: Bearer $TEAM_TOKEN`. Updates require a team token with write
permissions (see `authentication/brownie-authentication.yml`).

## Steps

1. **Read the effective config** — `GET /config/me/effective` returns the fully
   merged configuration: `mcp_servers`, `a2a_agents`, `agents` (per-agent
   `prompt` / `enabled` / `disable_default_tools` / `enable_extra_tools`),
   `feature_flags`, `knowledge_source`, and Slack routing fields.

   ```bash
   curl -X GET https://api.incidentfox.ai/api/v1/config/me/effective \
     -H "Authorization: Bearer $TEAM_TOKEN"
   ```

2. **Apply an override** — `PUT /config/me` with only the fields to change;
   values are deep-merged (PATCH semantics) into the existing config.

   ```bash
   curl -X PUT https://api.incidentfox.ai/api/v1/config/me \
     -H "Authorization: Bearer $TEAM_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"mcp_servers":["grafana","aws","coralogix","snowflake"]}'
   ```

3. **Read the response** — a `200` returns `message`, an incremented `version`,
   the `changed_fields[]`, and the new `effective_config`. When approval
   workflows are enabled the update returns `202` (approval required) instead
   of applying immediately.

## Conventions and error handling

- Errors use `{ "error": { "code", "message", "request_id" } }`; expect `401`,
  `403` (write without write permission), and `429` (writes capped at
  20/minute). See `errors/brownie-problem-types.yml` and
  `rate-limits/brownie-rate-limits.yml`.
