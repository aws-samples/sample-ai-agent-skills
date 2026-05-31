---
title: "F1 — Queue Configuration"
description: "Diagnose game session queue configuration issues"
status: active
severity: HIGH
triggers: ["queue error", "queue configuration", "queue not routing", "destination not found"]
owner: devops-agent
objective: "Resolve game session queue configuration and routing issues"
context: "Queues route game session placement requests to fleets. They support multi-fleet, multi-region destinations with priority and latency-based routing."
---

## Phase 1 — Triage

MUST:
- Check queue configuration: `aws gamelift describe-game-session-queues --names <queue-name>`
- Verify all destination fleets are ACTIVE: `aws gamelift describe-fleet-attributes --fleet-ids <fleet-ids>`
- Check queue metrics: `aws cloudwatch get-metric-statistics --namespace AWS/GameLift --metric-name PlacementsFailed --dimensions Name=QueueName,Value=<queue-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Verify destination fleet ARNs are correct and accessible
- Check if aliases are used and point to valid fleets
- Review player latency policies configuration

MAY:
- Check placement history for patterns: `aws gamelift describe-game-session-placement --placement-id <id>`
- Review queue timeout configuration

## Phase 2 — Remediate

MUST:
- Add valid fleet destinations: `aws gamelift update-game-session-queue --name <queue-name> --destinations <fleet-arns>`
- Ensure at least one destination fleet has available capacity
- Set appropriate placement timeout (default 600 seconds)

SHOULD:
- Configure priority settings for cost optimization (Spot first, On-Demand fallback)
- Add player latency policies for latency-sensitive games
- Use fleet aliases instead of direct fleet ARNs for easier fleet replacement

MAY:
- Create separate queues for different game modes or regions
- Implement custom placement event handling via SNS

## Safety Ratings

```
safety_ratings:
  - "describe-game-session-queues: GREEN — read-only inspection"
  - "update-game-session-queue: YELLOW — changes routing behavior"
  - "create-game-session-queue: YELLOW — creates new routing configuration"
  - "delete-game-session-queue: RED — stops all placements using this queue"
```

## Escalation Conditions

- Queue has no valid destinations (all fleets in ERROR or terminated)
- Placement consistently fails across all destinations
- Queue configuration changes require coordination with game client team
- Latency-based routing not working as expected

## Data Sensitivity

- `aws gamelift describe-game-session-queues`: LOW — queue configuration
- Queue destination list: LOW — fleet ARNs
- Placement data: MEDIUM — may contain player latency information
- Queue metrics: LOW — placement success/failure counts

## Prohibited Actions

- NEVER delete queues that are referenced by matchmaking configurations
- NEVER remove all destinations from a queue during active placements
- NEVER set placement timeout to 0
- NEVER modify production queues without testing in staging first

## Phase 3 — Rollback

1. If queue destinations were modified: restore original destination list
2. If queue was deleted: recreate with original configuration
3. If player latency policies were changed: restore original policies
4. If timeout was modified: restore original timeout value
5. Document queue changes and update matchmaking configuration if needed

## Common Issues

- symptoms: "Queue placements fail with 'No destinations available'"
  diagnosis: "All fleet destinations in the queue are either terminated, in ERROR state, or at maximum capacity."
  resolution: "Verify fleet status for all destinations. Scale up fleets or add new fleet destinations. Ensure at least one fleet is ACTIVE with available capacity."

## Output Format

```yaml
root_cause: "queue_configuration — <specific_cause>"
evidence:
  - type: queue_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix queue configuration"
  long_term: "Implement queue monitoring and automated fleet health checks"
```

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
