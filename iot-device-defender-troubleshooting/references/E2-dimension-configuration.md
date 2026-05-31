---
title: "E2 — Dimension Configuration"
description: "Diagnose dimension configuration issues for security profile scoping"
status: active
severity: MEDIUM
triggers: ["dimension not filtering", "wrong devices monitored", "topic filter error", "dimension configuration issue"]
owner: devops-agent
objective: "Resolve dimension configuration issues including topic filter patterns, security profile attachment, and scoping"
context: "Dimensions are topic filters that scope which MQTT topics a security profile behavior monitors. They allow fine-grained control over which device communications are analyzed."
---

## Phase 1 — Triage

MUST:
- List dimensions: `aws iot list-dimensions`
- Check dimension details: `aws iot describe-dimension --name <dimension-name>`
- Verify dimension is referenced in security profile behaviors: `aws iot describe-security-profile --security-profile-name <name>`

SHOULD:
- Verify topic filter pattern matches intended MQTT topics
- Check that devices publish to topics matching the dimension filter
- Review which behaviors use the dimension

MAY:
- Test topic filter pattern against sample MQTT topics
- Check if dimension is used in multiple security profiles

## Phase 2 — Remediate

MUST:
- Fix topic filter pattern to match intended MQTT topics
- Ensure dimension is referenced in the correct security profile behaviors
- Verify devices publish to topics that match the dimension filter

SHOULD:
- Use MQTT wildcards correctly (+ for single level, # for multi-level)
- Test dimension filter with a small set of devices before applying broadly

MAY:
- Create multiple dimensions for different device groups or topic patterns
- Document dimension naming conventions and filter patterns

## Safety Ratings

```
safety_ratings:
  - "iot list-dimensions: GREEN — read-only listing"
  - "iot describe-dimension: GREEN — read-only dimension details"
  - "iot update-dimension: YELLOW — modifies monitoring scope"
  - "iot update-security-profile (dimension reference): YELLOW — changes which topics are monitored"
```

## Escalation Conditions

- Dimension filter too broad causing excessive violation noise
- Dimension filter too narrow causing missed violations on critical topics
- Dimension changes affect multiple security profiles simultaneously
- Topic filter pattern syntax errors prevent dimension from working

## Data Sensitivity

- `aws iot list-dimensions`: LOW — returns dimension names
- `aws iot describe-dimension`: LOW — returns topic filter pattern
- `MQTT topic patterns`: MEDIUM — reveal device communication structure
- `security profile behavior configuration`: MEDIUM — contains monitoring rules

## Prohibited Actions

- NEVER create dimensions with overly broad topic filters (e.g., #) without understanding scope
- NEVER delete dimensions referenced by active security profiles
- NEVER modify dimension filters on production profiles without testing
- NEVER use dimensions to exclude critical device topics from monitoring

## Phase 3 — Rollback

1. If dimension was modified incorrectly, restore previous topic filter pattern
2. If dimension was deleted, recreate with original configuration
3. Update security profile behaviors to reference the correct dimension
4. Verify monitoring scope is correct by checking active violations
5. Document the dimension configuration and its purpose

## Common Issues

- symptoms: "Security profile not monitoring expected devices — dimension filter too restrictive"
  diagnosis: "Topic filter pattern does not match the MQTT topics devices publish to."
  resolution: "Update dimension topic filter to match device MQTT topic patterns."

## Output Format

```yaml
root_cause: "dimension_configuration — <specific_cause>"
evidence:
  - type: dimension_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix dimension filter"
  long_term: "Document and test dimension patterns before deployment"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
