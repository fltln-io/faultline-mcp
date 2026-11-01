# Faultline MCP Server

[Faultline](https://fltln.io) is infrastructure monitoring and incident
management for DevOps/SRE teams. This repo documents `faultline-mcp`,
Faultline's remote MCP server, which lets AI agents (Claude, Claude Code,
Claude Desktop, or anything MCP-compatible) operate Faultline: check
infrastructure health, inspect and act on incidents, look up who's on call,
and run approved runbooks.

**This repo is documentation plus an Agent Skill.** The server is hosted by
Faultline at `https://mcp.fltln.io/mcp` (Streamable HTTP transport), so
there's nothing to install or run yourself.

## Setup

1. Create an API key in Faultline: **Settings → API Keys**. Keys look like
   `flt_...`.
2. Point your MCP client at `https://mcp.fltln.io/mcp`, sending the key as
   either `X-API-Key: flt_...` or `Authorization: Bearer flt_...`.

**Claude Code:**

```bash
claude mcp add --transport http faultline-mcp https://mcp.fltln.io/mcp \
  --header "X-API-Key: flt_..."
```

**Clients that take raw JSON config** (Claude Desktop, etc.):

```json
{
  "mcpServers": {
    "faultline": {
      "type": "http",
      "url": "https://mcp.fltln.io/mcp",
      "headers": { "X-API-Key": "flt_..." }
    }
  }
}
```

### Discovery without a key

You can browse the toolset before creating a key. The MCP discovery methods
`initialize`, `notifications/initialized`, and `tools/list` work without an
API key, so connector directories, registry scanners, and curious humans can
see every tool's name, description, input schema, and annotations:

```bash
curl -s https://mcp.fltln.io/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

Anything else, including every `tools/call`, needs a key and returns `401`
without one.

## Agent Skill

[`skills/faultline/SKILL.md`](skills/faultline/SKILL.md) is an
[Agent Skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview)
that teaches Claude how to use these tools well:

- **Workflows:** which tools to call for a health overview, "why is X slow?",
  incident triage and remediation, and on-call lookups.
- **Approval rules:** read-only tools run freely; acknowledging, resolving,
  and running runbooks wait for your explicit go-ahead, and Claude only runs a
  runbook that `diagnose_incident` returned for that incident.

Install it alongside the MCP server:

- **Claude Code:** copy the `skills/faultline` folder into `~/.claude/skills/`
  (all projects) or `.claude/skills/` (one project).
- **Claude.ai / Claude Desktop:** zip the `faultline` folder and upload it
  under **Settings → Capabilities → Skills**.

## Tools

| Tool | What it does |
| --- | --- |
| `list_services` | Monitor inventory with current status (optional status filter) |
| `get_service` | One service + its 10 most recent checks (for diagnosis) |
| `list_incidents` | Open incidents (or `status: "resolved"` for history) |
| `get_incident` | Full incident record: timeline, AI summary, post-mortem |
| `acknowledge_incident` | Acknowledge an incident, which stops further escalation |
| `resolve_incident` | Resolve with an optional note (recorded on the timeline and visible on your status page) |
| `who_is_on_call` | Current on-call per schedule, with shift end time |
| `list_anomalies` | Recent learned-baseline latency anomalies (observed vs baseline, z-score, hours sustained, auto-opened incident if any) |
| `diagnose_incident` | Recommend the next action (run runbook / escalate / resolve / wait) + candidate runbooks. Analysis only; changes nothing |
| `run_runbook` | Execute one chosen runbook against an incident. **Mutates infrastructure** (can restart/scale services) |

## Security

- **Scoped to your API key.** The server never stores your key. It's used
  only for the duration of each request, proxied straight through to
  Faultline's API.
- **Keyless means read-only metadata.** Requests without a key can only list
  the static tool definitions. They never reach Faultline's API and can't
  read or change any tenant data.
- **Approval-gated mutation.** `run_runbook` is the only tool that changes
  infrastructure. Its description, and the Agent Skill, instruct the calling
  agent to use it only after `diagnose_incident` recommended it and you've
  explicitly confirmed.
- **Tenant-isolated.** Every request is scoped to the tenant that owns the
  API key, so one key can never see or affect another tenant's data.

## Support

Questions or issues: [support@fltln.io](mailto:support@fltln.io) or the
[Faultline dashboard](https://fltln.io).

[![MCP Badge](https://lobehub.com/badge/mcp/fltln-io-faultline-mcp)](https://lobehub.com/mcp/fltln-io-faultline-mcp)
