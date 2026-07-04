# Faultline MCP Server

[Faultline](https://fltln.io) is infrastructure monitoring and incident
management for DevOps/SRE teams. This repo documents `faultline-mcp` —
Faultline's remote MCP server, which lets AI agents (Claude, Claude Code,
Claude Desktop, or anything MCP-compatible) operate Faultline: check
infrastructure health, inspect and act on incidents, look up who's on call,
and run approved runbooks.

**This repo is documentation only.** The server is hosted by Faultline at
`https://mcp.fltln.io/mcp` (Streamable HTTP transport) — there's nothing to
install or run yourself.

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

## Tools

| Tool | What it does |
| --- | --- |
| `list_services` | Monitor inventory with current status (optional status filter) |
| `get_service` | One service + its 10 most recent checks (for diagnosis) |
| `list_incidents` | Open incidents (or `status: "resolved"` for history) |
| `get_incident` | Full incident record: timeline, AI summary, post-mortem |
| `acknowledge_incident` | Acknowledge an incident — stops further escalation |
| `resolve_incident` | Resolve with an optional note (recorded on the timeline) |
| `who_is_on_call` | Current on-call per schedule, with shift end time |
| `list_anomalies` | Recent learned-baseline latency anomalies (observed vs baseline, z-score, hours sustained, auto-opened incident if any) |
| `diagnose_incident` | Recommend the next action (run runbook / escalate / resolve / wait) + candidate runbooks. Analysis only — changes nothing |
| `run_runbook` | Execute one chosen runbook against an incident — **mutates infrastructure** (can restart/scale services) |

## Security

- **Scoped to your API key.** The server never stores your key — it's used
  only for the duration of each request, proxied straight through to
  Faultline's API.
- **Approval-gated mutation.** `run_runbook` is the only tool that changes
  infrastructure. Its description instructs the calling agent to use it only
  after `diagnose_incident` recommended it and you've explicitly confirmed.
- **Tenant-isolated.** Every request is scoped to the tenant that owns the
  API key — one key can never see or affect another tenant's data.

## Support

Questions or issues: [support@fltln.io](mailto:support@fltln.io) or the
[Faultline dashboard](https://fltln.io).
