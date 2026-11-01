---
name: faultline
description: Operate Faultline infrastructure monitoring through the Faultline MCP server. Use when the user asks what is down, why a service is slow or degraded, about open incidents, outages, latency spikes, who is on call, or asks to acknowledge, resolve, diagnose, or remediate an incident or run a runbook.
---

# Faultline

Faultline monitors HTTP endpoints, Docker containers, and ECS services, and opens incidents when they fail. The `faultline` MCP server (`https://mcp.fltln.io/mcp`) gives you ten tools over the user's live environment. If those tools are not available, tell the user to connect the server first (see Setup at the end).

## Ground rules

- **Read freely, act with permission.** Inspection tools change nothing; call them without asking. Before `acknowledge_incident`, `resolve_incident`, or `run_runbook`, say exactly what you are about to do and wait for an explicit yes, unless the user already asked for that specific action.
- **`run_runbook` mutates infrastructure.** It can restart or scale services. Only run a runbook that `diagnose_incident` returned (as the recommendation or a candidate) for that same incident, and only after the user confirms. Never invent or guess a `runbookId`.
- **`resolve_incident` is public.** It shows up on the customer's status page. Resolve only when the user confirms the underlying issue is fixed, and pass a short `note` describing the fix.
- **Use IDs from tool results.** Get `serviceId` and `incidentId` from `list_services` / `list_incidents`; don't ask the user to type them if you can look them up by name.
- **Report what the data says.** Quote status codes, latencies, z-scores, and timestamps from the results. If a tool returns an error such as `Faultline API 403`, tell the user (a 403 usually means their API key is scoped to a different service).

## Tools

| Tool | Changes anything? | Use it to |
|---|---|---|
| `list_services` | No | Get an overview of health, or find a service ID. Optional `status` filter: `OK`, `DEGRADED`, `DOWN`, `PAUSED`, `PENDING`. |
| `get_service` | No | See one service's last 10 checks (status codes, latency, errors) to explain *why* it is unhealthy. |
| `list_incidents` | No | Open incidents (default) or `status: "resolved"` for history. |
| `get_incident` | No | Full timeline, escalations, runbook results, AI summary, post-mortem. Read before acting. |
| `list_anomalies` | No | Latency anomalies against a learned baseline: early warnings before a hard failure. |
| `who_is_on_call` | No | Current on-call person per schedule and when their shift ends. |
| `diagnose_incident` | No | Recommendation (run runbook / escalate / resolve / wait) with rationale, confidence, and candidate runbooks. |
| `acknowledge_incident` | Yes | Stop escalation; the user is taking ownership. |
| `resolve_incident` | Yes, status-page visible | Close an incident with a fix note. |
| `run_runbook` | Yes, infrastructure | Execute one runbook against an incident. |

## Workflows

### "What's going on?" / health overview

1. `list_services` and `list_incidents` (in parallel).
2. If anything is `DOWN` or `DEGRADED`, or the user mentions slowness, also `list_anomalies`.
3. Summarize: what is broken, since when, which incidents are open, and anything trending bad. Lead with the worst problem. If everything is `OK` with no open incidents, say so in one line.

### "Why is X slow / degraded / down?"

1. Find the service with `list_services`, then `get_service` for its recent checks.
2. `list_anomalies` and look for entries on that service.
3. Explain the pattern (timeouts vs 5xx vs latency drift, when it started, sustained or flapping). If there is an open incident for it, mention it and offer to diagnose.

### Incident triage and remediation

1. `list_incidents` → `get_incident` for the one in question.
2. `diagnose_incident` and present the recommendation, its confidence, and rationale.
3. Act on the recommendation:
   - **Run runbook:** name the runbook and what it will do, ask for confirmation, then `run_runbook` with that `runbookId`. Afterwards, `get_incident` / `get_service` to check whether it worked.
   - **Escalate:** `who_is_on_call` and tell the user who to contact. Offer to acknowledge if the user is taking it.
   - **Wait:** explain what to watch for and offer to check again.
   - **Resolve:** confirm with the user that it's fixed, then `resolve_incident` with a note.
4. Never chain `diagnose_incident` → `run_runbook` → `resolve_incident` without the user's go-ahead at each mutating step.

### "Who's on call?"

`who_is_on_call`, and give the name per schedule with when the shift ends, in the user's terms ("until 5pm tomorrow"), not just a raw timestamp.

## Setup

The user needs a Faultline API key (`flt_...`) from **Settings → API Keys** in the Faultline dashboard. For Claude Code:

```bash
claude mcp add --transport http faultline https://mcp.fltln.io/mcp \
  --header "X-API-Key: flt_your_key_here"
```

Other clients: point them at `https://mcp.fltln.io/mcp` with an `X-API-Key` (or `Authorization: Bearer flt_...`) header. Full docs: https://fltln.io/docs/mcp
