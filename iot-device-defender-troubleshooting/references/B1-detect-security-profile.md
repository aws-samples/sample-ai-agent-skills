---
title: "B1 — Detect Security Profile"
description: "Diagnose security profile configuration and attachment issues"
status: active
severity: HIGH
triggers: ["security profile not working", "no violations detected", "profile attachment error", "behavior threshold issue"]
owner: devops-agent
objective: "Resolve security profile issues including behavior configuration, target attachment, and violation detection"
context: "Security profiles define expected device behaviors with thresholds. They must be attached to thing groups or all things. Violations are generated when device metrics exceed defined thresholds."
---

## Phase 1 — Triage

MUST:
- List security profiles: `aws iot list-security-profiles`
- Check profile details: `aws iot describe-security-profile --security-profile-name <name>`
- Check profile targets: `aws iot list-targets-for-security-profile --security-profile-name <name>`

SHOULD:
- Verify behaviors are correctly configured with appropriate metrics and thresholds
- Check active violations: `aws iot list-active-violations --security-profile-name <name>`
- Verify devices are reporting metrics to the Defender MQTT topic

MAY:
- Check violation history: `aws iot list-violation-events --start-time <start> --end-time <end> --security-profile-name <name>`
- Review device metric reports in CloudWatch

## Phase 2 — Remediate

MUST:
- Attach security profile to correct targets: `aws iot attach-security-profile --security-profile-name <name> --security-profile-target-arn <target-arn>`
- Fix behavior thresholds that are too strict (false positives) or too lenient (missed violations)
- Ensure devices are reporting metrics if using device-side metrics

SHOULD:
- Test profile with a small thing group before applying to all things
- Configure both statistical and absolute thresholds for comprehensive detection

MAY:
- Use dimensions to scope behaviors to specific MQTT topics
- Enable ML Detect for adaptive threshold learning

## Safety Ratings

```
safety_ratings:
  - "iot list-security-profiles: GREEN — read-only listing"
  - "iot describe-security-profile: GREEN — read-only profile details"
  - "iot attach-security-profile: YELLOW — starts monitoring devices, may generate violations"
  - "iot update-security-profile: YELLOW — modifies monitoring thresholds"
```

## Escalation Conditions

- Security profile generates excessive false positive violations
- Profile cannot be attached due to service quota limits
- Behavior metrics are not available for the target devices
- Profile changes cause widespread violation alerts affecting operations

## Data Sensitivity

- `aws iot list-security-profiles`: LOW — returns profile names
- `aws iot describe-security-profile`: MEDIUM — contains behavior thresholds and metric configurations
- `aws iot list-active-violations`: MEDIUM — reveals device security status
- `violation events`: MEDIUM — contain device identifiers and metric values

## Prohibited Actions

- NEVER delete a security profile monitoring production devices without a replacement
- NEVER set all thresholds to maximum values to suppress violations
- NEVER attach untested security profiles to all things in production
- NEVER ignore persistent violations — they may indicate compromised devices

## Phase 3 — Rollback

1. If profile was modified incorrectly, restore previous behavior configuration
2. If profile was attached to wrong targets, detach: `aws iot detach-security-profile --security-profile-name <name> --security-profile-target-arn <target-arn>`
3. Clear false positive violations by adjusting thresholds
4. Verify monitoring is restored for intended devices
5. Document the profile change and its impact

## Common Issues

- symptoms: "No violations detected despite known anomalous behavior"
  diagnosis: "Security profile not attached to any targets or devices not reporting metrics."
  resolution: "Attach profile to thing group or all things. Verify device metric reporting."

## Output Format

```yaml
root_cause: "security_profile — <specific_cause>"
evidence:
  - type: profile_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix security profile configuration"
  long_term: "Test profiles in staging before production deployment"
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
