# Z1 — Application Discovery Service General Troubleshooting

## When No Specific Runbook Matches

## General Diagnostic Commands

```bash
aws mgh describe-home-region
aws discovery describe-agents --max-results 20
aws discovery get-discovery-summary
aws discovery describe-continuous-exports --max-results 10
aws discovery describe-export-tasks --max-results 10
aws discovery list-configurations --configuration-type SERVER --max-results 20
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=discovery.amazonaws.com --max-results 20
```

## Escalation Checklist

1. Home region configuration
2. Agent count and health status
3. Agentless collector status (if VMware)
4. Data collection status (started/stopped)
5. Continuous export status
6. Discovery summary (server/process/connection counts)
7. CloudTrail events for recent changes
8. Timestamp of when issue started


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Check home region configuration | GREEN | Read-only check |
| Check agent count and health | GREEN | Read-only check |
| Check agentless collector status | GREEN | Read-only check |
| Check data collection status | GREEN | Read-only check |
| Check continuous export status | GREEN | Read-only check |
| Review CloudTrail events | GREEN | Read-only audit |

## Escalation Conditions

- Discovery agent deployed in production environment

## Data Sensitivity

- **Discovery summary**: Aggregate counts of servers, processes, connections — INTERNAL
- **Agent details**: Agent IDs, health status, host information — INTERNAL
- **CloudTrail events**: API call history for discovery service — INTERNAL

## Prohibited Actions

- NEVER deploy discovery agents without change management approval
- NEVER store IAM access keys in plain text on shared systems
- NEVER modify agent configuration on production servers without a rollback plan

## Phase 3 — Rollback

1. Stop all data collection: `aws discovery stop-data-collection-by-agent-ids --agent-ids <all-agent-ids>`
2. Stop continuous export if running: `aws discovery stop-continuous-export --export-id <id>`
3. Revert any configuration changes identified in CloudTrail
4. Verify all agents are in SHUTDOWN state if decommissioning
5. Document the issue and resolution for future reference

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
