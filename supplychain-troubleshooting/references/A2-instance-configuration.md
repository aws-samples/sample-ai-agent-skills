---
title: "A2 — Instance Configuration"
description: "Diagnose Supply Chain instance configuration issues"
status: active
severity: MEDIUM
triggers: ["instance configuration", "instance settings", "instance state error"]
owner: devops-agent
objective: "Resolve Supply Chain instance configuration issues"
context: "Instances have limited mutable configuration. Some settings are immutable after creation."
---

## Phase 1 — Triage

MUST:
- Get instance details: `aws scn get-instance --instance-id <instance-id>`
- Verify instance state is ACTIVE
- Check instance tags: `aws scn list-tags-for-resource --resource-arn <instance-arn>`

SHOULD:
- Verify KMS encryption configuration if applicable
- Check associated S3 bucket configuration

MAY:
- Review instance creation history in CloudTrail

## Phase 2 — Remediate

MUST:
- If instance is in FAILED state: delete and recreate with correct configuration
- Fix associated resource configurations (S3 buckets, KMS keys)

SHOULD:
- Update tags for proper resource management: `aws scn tag-resource --resource-arn <arn> --tags Key=Environment,Value=Production`

MAY:
- Set up CloudWatch alarms for instance health

## Safety Ratings

```
safety_ratings:
  - "get-instance: GREEN — read-only instance details"
  - "list-tags-for-resource: GREEN — read-only tag enumeration"
  - "tag-resource: GREEN — adds metadata tags, non-destructive"
  - "delete-instance: RED — permanently removes instance and all data"
```

## Escalation Conditions

- Instance stuck in non-ACTIVE state for extended period
- Configuration change requires instance recreation with data migration
- KMS key used by instance is compromised or scheduled for deletion
- Instance needs to be migrated to a different region

## Data Sensitivity

- `aws scn get-instance`: LOW — returns configuration metadata
- `aws scn list-tags-for-resource`: LOW — returns tag metadata
- Instance data: HIGH — may contain proprietary supply chain data

## Prohibited Actions

- NEVER delete an instance without confirming all data has been exported
- NEVER modify instance configuration during active planning operations
- NEVER assume instance configuration can be changed for immutable settings
- NEVER modify tags used for compliance tracking without approval

## Phase 3 — Rollback

1. If tags were modified incorrectly: restore original tags
2. If associated S3 bucket configuration was changed: revert bucket settings
3. If an instance was deleted prematurely: data cannot be recovered — escalate
4. Document all configuration changes and notify the operations team
5. Verify instance is functioning normally after any changes

## Common Issues

- symptoms: "Instance state is DELETE_FAILED"
  diagnosis: "Instance deletion failed due to remaining resources."
  resolution: "Remove all data integration flows and datasets, then retry deletion."

- symptoms: "Instance operations return throttling errors"
  diagnosis: "API request rate exceeds service quotas."
  resolution: "Implement exponential backoff. Request quota increase if sustained high throughput is needed."

## Output Format

```yaml
root_cause: "instance_configuration — <specific_cause>"
evidence:
  - type: configuration_issue
    content: "<details>"
severity: MEDIUM
mitigation:
  immediate: "Fix configuration issue"
  long_term: "Implement configuration monitoring"
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
