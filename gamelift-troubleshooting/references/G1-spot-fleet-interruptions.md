---
title: "G1 — Spot Fleet Interruptions"
description: "Diagnose GameLift Spot fleet interruption and fallback issues"
status: active
severity: HIGH
triggers: ["Spot interruption", "Spot reclaimed", "instance terminated", "Spot capacity", "fleet interrupted"]
owner: devops-agent
objective: "Resolve Spot fleet interruption handling and On-Demand fallback issues"
context: "Spot instances can be reclaimed by AWS with 2-minute warning. Game servers must handle SIGTERM gracefully. Queues should include On-Demand fleets as fallback."
---

## Phase 1 — Triage

MUST:
- Check fleet type (SPOT vs ON_DEMAND): `aws gamelift describe-fleet-attributes --fleet-ids <fleet-id> --query 'FleetAttributes[0].FleetType'`
- Check fleet events for interruptions: `aws gamelift describe-fleet-events --fleet-id <fleet-id> --start-time <start> --end-time <end>`
- Verify queue has On-Demand fallback: `aws gamelift describe-game-session-queues --names <queue-name>`

SHOULD:
- Check Spot instance availability in the region
- Verify game server handles SIGTERM signal for graceful shutdown
- Check CloudWatch for interruption frequency: `aws cloudwatch get-metric-statistics --namespace AWS/GameLift --metric-name InstanceInterruptions --dimensions Name=FleetId,Value=<fleet-id> --start-time <start> --end-time <end> --period 3600 --statistics Sum`

MAY:
- Review Spot pricing history for the instance type
- Check if multiple instance types are configured for Spot diversity

## Phase 2 — Remediate

MUST:
- Ensure queue includes On-Demand fleet as fallback destination
- Implement SIGTERM handler in game server for graceful shutdown
- Configure queue priority: Spot fleets first, On-Demand as fallback

SHOULD:
- Use multiple instance types for Spot fleets to increase availability
- Implement game state saving on SIGTERM for session recovery
- Set up CloudWatch alarms for high interruption rates

MAY:
- Use Spot fleet with multiple instance type preferences
- Implement player notification system for Spot interruptions
- Consider On-Demand for competitive/ranked game modes

## Safety Ratings

```
safety_ratings:
  - "describe-fleet-attributes / describe-fleet-events: GREEN — read-only inspection"
  - "describe-game-session-queues: GREEN — read-only queue inspection"
  - "update-game-session-queue (add On-Demand): YELLOW — changes routing, adds cost"
  - "create-fleet (On-Demand fallback): YELLOW — creates instances, higher cost than Spot"
```

## Escalation Conditions

- Spot capacity consistently unavailable in the region
- Game server does not handle SIGTERM (requires code changes)
- On-Demand fallback fleet at capacity during Spot interruptions
- Player experience severely impacted by frequent interruptions

## Data Sensitivity

- `aws gamelift describe-fleet-attributes`: LOW — fleet metadata
- `aws gamelift describe-fleet-events`: LOW — fleet lifecycle events
- Spot pricing data: LOW — public pricing information
- Game session state during interruption: HIGH — may contain player game data

## Prohibited Actions

- NEVER use Spot-only fleets without On-Demand fallback for production
- NEVER ignore SIGTERM signals in game servers on Spot fleets
- NEVER assume Spot capacity will always be available
- NEVER use Spot fleets for game modes where interruption is unacceptable

## Phase 3 — Rollback

1. If Spot fleet was the only destination: add On-Demand fleet to queue immediately
2. If game server SIGTERM handler was broken: deploy fixed build to new fleet
3. If queue priority was misconfigured: restore original priority settings
4. If On-Demand fallback fleet was removed: recreate and add to queue
5. Document interruption patterns and update capacity planning

## Common Issues

- symptoms: "Players are disconnected mid-game with no warning on Spot fleet"
  diagnosis: "Game server does not handle SIGTERM signal sent 2 minutes before Spot interruption."
  resolution: "Implement SIGTERM handler in game server to save game state and notify players. Add On-Demand fleet to queue as fallback."

## Output Format

```yaml
root_cause: "spot_interruption — <specific_cause>"
evidence:
  - type: spot_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Ensure On-Demand fallback and graceful shutdown"
  long_term: "Implement Spot interruption handling and multi-instance-type Spot fleets"
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
