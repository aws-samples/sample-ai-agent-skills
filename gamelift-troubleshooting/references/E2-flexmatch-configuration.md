---
title: "E2 — FlexMatch Configuration"
description: "Diagnose FlexMatch matchmaking configuration issues"
status: active
severity: MEDIUM
triggers: ["FlexMatch config", "rule set error", "matchmaking configuration", "expansion rules"]
owner: devops-agent
objective: "Resolve FlexMatch configuration, rule set, and expansion rule issues"
context: "FlexMatch uses rule sets to define matching criteria. Configurations link rule sets to queues. Expansion rules relax criteria over time. Two modes: standalone and integrated."
---

## Phase 1 — Triage

MUST:
- Check configuration: `aws gamelift describe-matchmaking-configurations --names <config-name>`
- Validate rule set: `aws gamelift validate-matchmaking-rule-set --rule-set-body '<rule-set-json>'`
- Check rule set details: `aws gamelift describe-matchmaking-rule-sets --names <rule-set-name>`

SHOULD:
- Verify queue exists and has capacity (for integrated mode)
- Check if expansion rules are configured for small player pools
- Review matchmaking metrics for success/failure rates

MAY:
- Test rule set with sample player data
- Check SNS notification configuration for matchmaking events

## Phase 2 — Remediate

MUST:
- Fix rule set JSON syntax: ensure valid FlexMatch rule set format
- Create or update configuration: `aws gamelift create-matchmaking-configuration --name <name> --rule-set-name <rule-set> --game-session-queue-arns <queue-arn> --request-timeout-seconds 300`
- For standalone mode: omit game-session-queue-arns

SHOULD:
- Add expansion rules to handle varying player pool sizes
- Set appropriate acceptance timeout for manual acceptance mode
- Configure backfill mode for partially filled game sessions

MAY:
- Create multiple configurations for different game modes
- Implement custom event handling via SNS notifications

## Safety Ratings

```
safety_ratings:
  - "describe-matchmaking-configurations / describe-matchmaking-rule-sets: GREEN — read-only inspection"
  - "validate-matchmaking-rule-set: GREEN — validates without creating"
  - "create-matchmaking-configuration: YELLOW — creates new matchmaking config"
  - "delete-matchmaking-configuration: RED — stops matchmaking for this config"
```

## Escalation Conditions

- Rule set validation passes but matchmaking consistently fails
- Expansion rules not relaxing criteria as expected
- Queue capacity insufficient for matched groups
- Complex rule set requirements exceed FlexMatch capabilities

## Data Sensitivity

- `aws gamelift describe-matchmaking-configurations`: LOW — configuration metadata
- Rule set content: LOW — matching criteria definitions
- Player attributes used in matching: MEDIUM — skill ratings, latency data
- Matchmaking event notifications: MEDIUM — match results and player groupings

## Prohibited Actions

- NEVER delete rule sets referenced by active configurations
- NEVER create rule sets with impossible matching criteria
- NEVER set request timeout below 10 seconds
- NEVER expose matchmaking internals (skill ratings, algorithms) to players

## Phase 3 — Rollback

1. If configuration was modified: restore original settings
2. If rule set was changed: create new rule set with original rules and update config
3. If expansion rules were removed: restore original expansion configuration
4. If acceptance timeout was changed: restore original timeout
5. Document configuration changes and monitor match quality metrics

## Common Issues

- symptoms: "validate-matchmaking-rule-set returns error: 'Invalid rule set'"
  diagnosis: "Rule set JSON has syntax errors or uses unsupported rule types."
  resolution: "Validate JSON syntax. Ensure rule types are valid (distance, comparison, collection, latency). Check FlexMatch rule set documentation for correct format."

## Output Format

```yaml
root_cause: "flexmatch_configuration — <specific_cause>"
evidence:
  - type: config_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix FlexMatch configuration"
  long_term: "Implement rule set testing and validation in CI/CD pipeline"
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
