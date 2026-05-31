# C3 — Application Discovery Performance Data Issues

## Symptoms
- No performance metrics for discovered servers
- CPU/memory data missing
- Performance data incomplete

## Triage Commands

```bash
aws discovery list-configurations --configuration-type SERVER --max-results 20
aws discovery describe-agents --max-results 20
aws discovery get-discovery-summary
```

## Common Causes
1. **Agent not collecting** — Data collection not started
2. **Insufficient collection time** — Need hours for meaningful metrics
3. **Agent permissions** — Needs elevated privileges for performance data
4. **Agentless limitations** — vCenter provides limited performance data
5. **Data retention** — Old performance data may have expired

## Resolution
- Verify data collection is started for agents
- Allow at least 24 hours for representative performance data
- Run agent with root/administrator for full performance metrics
- For agentless: performance data comes from vCenter — limited to what vCenter tracks
- Export performance data to S3 via continuous export for long-term retention


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Verify data collection is started | GREEN | Read-only check |
| Allow 24 hours for data | GREEN | Passive wait |
| Run agent with root/admin | YELLOW | Elevated privilege requirement |
| Export performance data to S3 | GREEN | Data export, non-destructive |

## Escalation Conditions

- Discovery agent deployed in production environment

## Data Sensitivity

- **Performance metrics**: CPU, memory, disk utilization — INTERNAL
- **Server resource profiles**: Capacity and usage patterns — INTERNAL
- **Exported data in S3**: Aggregated performance data — INTERNAL

## Prohibited Actions

- NEVER deploy discovery agents without change management approval
- NEVER store IAM access keys in plain text on shared systems
- NEVER modify agent configuration on production servers without a rollback plan

## Phase 3 — Rollback

1. Stop data collection: `aws discovery stop-data-collection-by-agent-ids --agent-ids <ids>`
2. Revoke elevated privileges if granted for performance data
3. Stop continuous export if enabled: `aws discovery stop-continuous-export --export-id <id>`
4. Delete exported performance data from S3 if required
5. Verify collection has stopped via agent status check

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
