# A2 — Application Discovery Agent Health Issues

## Symptoms
- Agent status UNHEALTHY or SHUTDOWN
- Agent not collecting data
- Agent connectivity lost

## Triage Commands

```bash
aws discovery describe-agents --filters name=agentIds,values=<agent-id>,condition=EQUALS
aws discovery describe-agents --filters name=health,values=UNHEALTHY,condition=EQUALS
```

## Agent Health States

| State | Meaning |
|-------|---------|
| HEALTHY | Agent running and reporting |
| UNHEALTHY | Agent running but can't communicate |
| RUNNING | Agent active, collection may or may not be started |
| SHUTDOWN | Agent stopped |
| BLACKLISTED | Agent blocked by service |

## Common Causes

1. **Network connectivity lost** — Outbound HTTPS blocked or DNS failure
2. **Agent service stopped** — OS-level service not running
3. **Credentials expired** — IAM credentials used by agent expired
4. **Disk full** — Agent can't write local buffer
5. **Agent version outdated** — Old version incompatible

## Resolution

- Check agent service status on the server (systemctl/service commands)
- Verify network connectivity to discovery endpoint
- Update IAM credentials if expired
- Check disk space on agent server
- Update agent to latest version


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Check agent service status | GREEN | Read-only diagnostic |
| Verify network connectivity | GREEN | Read-only check |
| Update IAM credentials | YELLOW | Credential rotation on host |
| Check disk space | GREEN | Read-only check |
| Update agent to latest version | YELLOW | Software update on server |

## Escalation Conditions

- Discovery agent deployed in production environment

## Data Sensitivity

- **Agent health data**: Status and connectivity info — INTERNAL
- **Server credentials**: IAM keys on agent hosts — CONFIDENTIAL
- **Disk and system info**: Server resource details — INTERNAL

## Prohibited Actions

- NEVER deploy discovery agents without change management approval
- NEVER store IAM access keys in plain text on shared systems
- NEVER modify agent configuration on production servers without a rollback plan

## Phase 3 — Rollback

1. Stop data collection for the agent: `aws discovery stop-data-collection-by-agent-ids --agent-ids <id>`
2. Revert agent to previous version if update caused issues
3. Restore previous IAM credentials if credential rotation caused failure
4. Restart agent service to previous known-good state
5. Verify agent health returns to HEALTHY status

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
