# E1 — Dependency Mapping Issues

## Symptoms
- No dependencies shown between servers
- Incomplete dependency graph
- Missing application dependencies

## Triage Commands

```bash
aws discovery list-configurations --configuration-type CONNECTION --max-results 50
aws discovery list-configurations --configuration-type PROCESS --max-results 50
aws discovery get-discovery-summary
```

## Common Causes

1. **Agentless only** — Dependencies require agent-based discovery
2. **Short collection period** — Need weeks for complete picture
3. **Agent not root** — Elevated privileges needed for network data
4. **Internal-only connections** — Localhost connections filtered out
5. **Agents not on all servers** — Missing agents = missing connections

## Resolution

- Deploy agents on ALL servers in the application stack
- Run agents with root/administrator privileges
- Collect for minimum 2 weeks for representative data
- Use Athena queries on continuous export for detailed analysis
- Network connections show: source, destination, port, protocol, bytes
- Combine with process data for application-level dependency mapping


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Deploy agents on all servers | YELLOW | Software installation across fleet |
| Run agents with root/admin | YELLOW | Elevated privilege requirement |
| Collect for minimum 2 weeks | GREEN | Extended passive collection |
| Use Athena queries for analysis | GREEN | Read-only data analysis |

## Escalation Conditions

- Discovery agent deployed in production environment

## Data Sensitivity

- **Dependency maps**: Application communication topology — CONFIDENTIAL
- **Network connections**: Source/destination/port data — CONFIDENTIAL
- **Process data**: Running applications and services — INTERNAL

## Prohibited Actions

- NEVER deploy discovery agents without change management approval
- NEVER store IAM access keys in plain text on shared systems
- NEVER modify agent configuration on production servers without a rollback plan

## Phase 3 — Rollback

1. Stop data collection on all agents: `aws discovery stop-data-collection-by-agent-ids --agent-ids <ids>`
2. Uninstall agents from servers where they are no longer needed
3. Revoke elevated privileges granted for dependency mapping
4. Stop continuous export: `aws discovery stop-continuous-export --export-id <id>`
5. Clean up Athena tables and S3 data if required

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
