---
title: "B2 — Matchmaking Failures"
description: "Diagnose FlexMatch matchmaking failures"
status: active
severity: HIGH
triggers: ["matchmaking failed", "no match found", "FlexMatch error", "matchmaking timeout", "ticket failed"]
owner: devops-agent
objective: "Resolve FlexMatch matchmaking configuration and ticket failures"
context: "FlexMatch matches players based on rules. It can work standalone or integrated with GameLift queues. Matchmaking has configurable timeout and expansion rules."
---

## Phase 1 — Triage

MUST:
- Check matchmaking configuration: `aws gamelift describe-matchmaking-configurations --names <config-name>`
- Check matchmaking ticket status: `aws gamelift describe-matchmaking --ticket-ids <ticket-id>`
- Verify rule set: `aws gamelift describe-matchmaking-rule-sets --names <rule-set-name>`

SHOULD:
- Check matchmaking metrics: `aws cloudwatch get-metric-statistics --namespace AWS/GameLift --metric-name MatchmakingSucceeded --dimensions Name=MatchmakingConfigurationName,Value=<config-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Verify game session queue exists and has capacity (for integrated mode)

MAY:
- Review matchmaking event notifications via SNS
- Check player pool size and distribution

## Phase 2 — Remediate

MUST:
- Fix rule set syntax errors: validate JSON rule set format
- Ensure matchmaking configuration points to valid queue (integrated mode)
- Set appropriate timeout and expansion rules for player pool size

SHOULD:
- Add expansion rules to relax criteria over time for small player pools
- Configure SNS notifications for matchmaking events
- Test rule set with sample player attributes

MAY:
- Use backfill to fill partially empty game sessions
- Implement custom matchmaking event handling

## Safety Ratings

```
safety_ratings:
  - "describe-matchmaking-configurations / describe-matchmaking: GREEN — read-only inspection"
  - "describe-matchmaking-rule-sets: GREEN — read-only rule inspection"
  - "update-matchmaking-configuration: YELLOW — changes matchmaking behavior"
  - "delete-matchmaking-configuration: RED — stops all matchmaking for this config"
```

## Escalation Conditions

- Matchmaking consistently times out due to insufficient player pool
- Rule set too restrictive for the player base size
- Game session queue has no capacity for matched groups
- Matchmaking latency exceeds acceptable thresholds

## Data Sensitivity

- `aws gamelift describe-matchmaking-configurations`: LOW — configuration metadata
- `aws gamelift describe-matchmaking`: MEDIUM — may contain player attributes
- Matchmaking rule sets: LOW — matching criteria
- Player attributes: MEDIUM — may contain skill ratings or player data

## Prohibited Actions

- NEVER delete matchmaking configurations during active matchmaking
- NEVER set matchmaking timeout to less than 10 seconds
- NEVER create rule sets that guarantee no matches (impossible criteria)
- NEVER expose player attributes or matchmaking data externally

## Phase 3 — Rollback

1. If matchmaking configuration was modified: restore original settings
2. If rule set was changed: create new rule set with original rules and update config
3. If queue was removed from configuration: re-associate the queue
4. If expansion rules were removed: restore original expansion configuration
5. Document matchmaking changes and monitor match success rates

## Common Issues

- symptoms: "Matchmaking tickets consistently time out with no match found"
  diagnosis: "Player pool is too small or rule set is too restrictive for the available players."
  resolution: "Add expansion rules to relax criteria over time. Increase matchmaking timeout. Consider reducing team size requirements."

## Output Format

```yaml
root_cause: "matchmaking_failure — <specific_cause>"
evidence:
  - type: matchmaking_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix matchmaking configuration"
  long_term: "Tune rule sets and expansion rules based on player pool analytics"
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
