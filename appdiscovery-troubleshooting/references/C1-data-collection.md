# C1 — Data Collection Issues

## Symptoms
- No data appearing after starting collection
- Data collection stops unexpectedly
- Incomplete server data

## Triage Commands

```bash
aws discovery describe-agents --max-results 20
aws discovery get-discovery-summary
aws discovery list-configurations --configuration-type SERVER --max-results 20
aws discovery list-configurations --configuration-type PROCESS --max-results 20
```

## Common Causes

1. **Collection not started** — Must explicitly start after agent registration
2. **Agent unhealthy** — Agent can't communicate with service
3. **Insufficient permissions** — Agent needs root/admin for full data
4. **Collection stopped** — Someone stopped collection via API
5. **Data delay** — Initial data takes 15-60 minutes to appear

## Resolution

- Start collection: `aws discovery start-data-collection-by-agent-ids --agent-ids <id1> <id2>`
- Verify agent health: look for HEALTHY status
- Run agent with root/administrator privileges for complete data
- Check collection status: agents should show `collectionStatus: STARTED`
- Wait 15-60 minutes for initial data to appear in console


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Start data collection | YELLOW | Begins collecting server data |
| Verify agent health | GREEN | Read-only check |
| Run agent with root/admin privileges | YELLOW | Elevated privilege requirement |
| Check collection status | GREEN | Read-only check |
| Wait for initial data | GREEN | Passive wait |

## Escalation Conditions

- Discovery agent deployed in production environment

## Data Sensitivity

- **Server data**: CPU, memory, OS, processes — INTERNAL
- **Process data**: Running applications and services — INTERNAL
- **Collection credentials**: Agent IAM credentials — CONFIDENTIAL

## Prohibited Actions

- NEVER deploy discovery agents without change management approval
- NEVER store IAM access keys in plain text on shared systems
- NEVER modify agent configuration on production servers without a rollback plan

## Phase 3 — Rollback

1. Stop data collection: `aws discovery stop-data-collection-by-agent-ids --agent-ids <id1> <id2>`
2. Verify collection has stopped by checking agent status
3. Remove elevated privileges from agent if they were granted
4. Delete collected data if required by data governance policies
5. Document the rollback and notify stakeholders

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
