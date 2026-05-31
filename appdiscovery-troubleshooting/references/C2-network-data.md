# C2 — Network Data Collection Issues

## Symptoms
- No network connection data
- Dependency map empty
- Missing connections between servers

## Triage Commands

```bash
aws discovery list-configurations --configuration-type CONNECTION --max-results 20
aws discovery describe-agents --filters name=agentIds,values=<agent-id>,condition=EQUALS
```

## Common Causes

1. **Using agentless** — Network connections only collected by agents
2. **Agent not root** — Needs elevated privileges for network data
3. **Collection too short** — Need days/weeks for complete dependency picture
4. **Firewall data** — Agent captures OS-level connections, not firewall logs
5. **Ephemeral connections** — Short-lived connections may be missed

## Resolution

- Deploy Discovery Agents (not agentless) for network data
- Run agents with root/administrator privileges
- Collect for at least 1-2 weeks for representative dependency data
- Network data includes: source IP, destination IP, port, bytes transferred
- Use continuous export + Athena for detailed network analysis


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Deploy Discovery Agents | YELLOW | Software installation on servers |
| Run agents with root/admin | YELLOW | Elevated privilege requirement |
| Collect for 1-2 weeks | GREEN | Extended passive collection |
| Use continuous export + Athena | GREEN | Read-only data analysis |

## Escalation Conditions

- Discovery agent deployed in production environment

## Data Sensitivity

- **Network connections**: Source/destination IPs, ports, bytes — CONFIDENTIAL
- **Dependency maps**: Application communication patterns — INTERNAL
- **Process-to-connection mapping**: Application network behavior — INTERNAL

## Prohibited Actions

- NEVER deploy discovery agents without change management approval
- NEVER store IAM access keys in plain text on shared systems
- NEVER modify agent configuration on production servers without a rollback plan

## Phase 3 — Rollback

1. Stop data collection on all agents: `aws discovery stop-data-collection-by-agent-ids --agent-ids <ids>`
2. Remove agents from servers if no longer needed
3. Revoke elevated privileges granted for network data collection
4. Stop continuous export if running: `aws discovery stop-continuous-export --export-id <id>`
5. Clean up exported data in S3 and Athena if required

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
