---
title: "C2 — Lifecycle Policy Issues"
description: "Diagnose MediaStore lifecycle policy configuration and execution issues"
status: active
severity: HIGH
triggers: ["lifecycle policy error", "objects not deleted", "retention policy", "lifecycle not working"]
owner: devops-agent
objective: "Resolve MediaStore lifecycle policy configuration and execution issues"
context: "Lifecycle policies automatically delete objects after a specified retention period. Deletion is permanent — there is no versioning or recovery."
---

## Phase 1 — Triage

MUST:
- Get current lifecycle policy: `aws mediastore get-lifecycle-policy --container-name <name>`
- Verify the policy JSON syntax and rule structure
- Check if the policy targets the correct path prefix

SHOULD:
- Verify objects are being deleted as expected: `aws mediastore-data list-items --endpoint <endpoint> --path /`
- Check CloudTrail for lifecycle-related events

MAY:
- Compare object timestamps with the lifecycle retention period

## Phase 2 — Remediate

MUST:
- Fix lifecycle policy syntax: `aws mediastore put-lifecycle-policy --container-name <name> --lifecycle-policy file://lifecycle.json`
- Verify the rule applies to the correct path prefix
- Confirm the retention period (DaysAfterCreation) is correct

SHOULD:
- Test lifecycle policy on a non-production container first
- Set up CloudWatch alarms for unexpected object count changes

MAY:
- Use multiple rules for different path prefixes with different retention periods

## Safety Ratings

```
safety_ratings:
  - "mediastore get-lifecycle-policy: GREEN — read-only policy retrieval"
  - "mediastore put-lifecycle-policy: RED — may cause permanent data deletion"
  - "mediastore delete-lifecycle-policy: YELLOW — stops automatic deletion"
  - "mediastore-data list-items: GREEN — read-only object listing"
```

## Escalation Conditions

- Lifecycle policy deletes objects that should be retained
- Lifecycle policy is not executing (objects past retention are not deleted)
- Need to recover objects deleted by lifecycle policy (not possible without backup)
- Multiple lifecycle rules conflict with each other

## Data Sensitivity

- Lifecycle policy document: LOW — contains retention rules
- Object listing: LOW — paths and metadata
- Retention periods: LOW — numeric values
- CloudTrail events: MEDIUM — reveals deletion actions

## Prohibited Actions

- NEVER apply lifecycle policies to production containers without testing
- NEVER set retention periods shorter than the live streaming window
- NEVER assume deleted objects can be recovered — there is no versioning
- NEVER delete the lifecycle policy to stop deletions without understanding the storage cost impact

## Phase 3 — Rollback

1. Remove lifecycle policy to stop deletions: `aws mediastore delete-lifecycle-policy --container-name <name>`
2. Objects already deleted cannot be recovered
3. Re-apply corrected lifecycle policy after testing
4. Verify remaining objects are not affected
5. Document the data loss if objects were incorrectly deleted

## Common Issues

- symptoms: "Lifecycle policy set but objects are not being deleted after retention period"
  diagnosis: "The path prefix in the rule does not match the actual object paths."
  resolution: "Verify the rule's path prefix matches object paths. Paths are case-sensitive."

- symptoms: "Objects deleted sooner than expected"
  diagnosis: "DaysAfterCreation is calculated from the object creation time, not the last modified time."
  resolution: "Adjust DaysAfterCreation to account for the full retention requirement."

## Output Format

```yaml
root_cause: "lifecycle — <specific_cause>"
evidence:
  - type: lifecycle_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix or remove lifecycle policy"
  long_term: "Test lifecycle policies in staging before production"
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
