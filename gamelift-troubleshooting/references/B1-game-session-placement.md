---
title: "B1 — Game Session Placement"
description: "Diagnose game session placement failures"
status: active
severity: HIGH
triggers: ["placement failed", "cannot create game session", "queue placement timeout", "no available capacity"]
owner: devops-agent
objective: "Resolve game session placement and queue-based session creation issues"
context: "Game sessions can be created directly on a fleet or via queues. Queues provide multi-fleet/multi-region failover. Placement failures are usually capacity issues."
---

## Phase 1 — Triage

MUST:
- Check queue configuration: `aws gamelift describe-game-session-queues --names <queue-name>`
- Check placement status: `aws gamelift describe-game-session-placement --placement-id <placement-id>`
- Verify fleet capacity in queue destinations: `aws gamelift describe-fleet-capacity --fleet-id <fleet-id>`

SHOULD:
- Check active game sessions on target fleets: `aws gamelift describe-game-sessions --fleet-id <fleet-id> --status-filter ACTIVE`
- Verify fleet status is ACTIVE: `aws gamelift describe-fleet-attributes --fleet-ids <fleet-id>`

MAY:
- Review CloudWatch placement metrics: `aws cloudwatch get-metric-statistics --namespace AWS/GameLift --metric-name PlacementsFailed --dimensions Name=QueueName,Value=<queue-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check player latency data if using latency-based placement

## Phase 2 — Remediate

MUST:
- Ensure at least one fleet in the queue has available capacity
- Scale up fleets if all are at capacity: `aws gamelift update-fleet-capacity --fleet-id <id> --desired-instances <count>`
- Verify queue destinations point to ACTIVE fleets

SHOULD:
- Add multiple fleets to the queue for failover
- Configure player latency policies for optimal placement
- Set appropriate placement timeout

MAY:
- Add fleets in additional regions for geographic distribution
- Use priority-based placement for cost optimization (Spot first, On-Demand fallback)

## Safety Ratings

```
safety_ratings:
  - "describe-game-session-queues / describe-game-session-placement: GREEN — read-only inspection"
  - "describe-fleet-capacity: GREEN — read-only capacity check"
  - "update-fleet-capacity: YELLOW — changes instance count"
  - "start-game-session-placement: GREEN — requests session placement"
```

## Escalation Conditions

- All fleets in queue at maximum capacity
- Placement consistently times out across all destinations
- Player latency requirements cannot be met in available regions
- Queue configuration changes require coordination across teams

## Data Sensitivity

- `aws gamelift describe-game-session-queues`: LOW — queue configuration
- `aws gamelift describe-game-session-placement`: MEDIUM — may contain player data
- `aws gamelift describe-fleet-capacity`: LOW — instance counts
- Player latency data: LOW — network latency measurements

## Prohibited Actions

- NEVER remove all destinations from a queue during active placements
- NEVER set placement timeout to 0 (use reasonable timeout like 120-600 seconds)
- NEVER bypass queues for production traffic (use direct CreateGameSession only for testing)
- NEVER ignore placement failures — they indicate capacity or configuration issues

## Phase 3 — Rollback

1. If queue destinations were modified: restore original destination list
2. If fleet was scaled incorrectly: restore original desired instance count
3. If player latency policy was changed: restore original policy
4. If placement timeout was modified: restore original timeout value
5. Document placement issues and update capacity planning

## Common Issues

- symptoms: "Game session placement times out with TIMED_OUT status"
  diagnosis: "No fleet in the queue has available capacity for a new game session."
  resolution: "Scale up fleets in the queue. Add more fleets as destinations. Increase max instances in scaling configuration."

## Output Format

```yaml
root_cause: "game_session_placement — <specific_cause>"
evidence:
  - type: placement_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Restore placement capacity"
  long_term: "Implement auto-scaling and multi-fleet queues for resilient placement"
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
