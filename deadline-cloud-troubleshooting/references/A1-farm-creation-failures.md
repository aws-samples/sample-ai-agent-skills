---
title: "A1 — Farm Creation Failures"
description: "Diagnose Deadline Cloud farm creation failures"
status: active
severity: HIGH
triggers: ["cannot create farm", "farm creation failed", "CreateFarm error"]
owner: devops-agent
objective: "Resolve farm creation issues"
context: "Farms are the top-level container in Deadline Cloud. They contain fleets and queues. Farm creation requires IAM permissions and optionally a KMS key for encryption."
---

## Phase 1 — Triage

MUST:
- Check existing farms: `aws deadline list-farms`
- Verify IAM permissions for deadline:CreateFarm
- Check if KMS key is accessible (if specified): `aws kms describe-key --key-id <key-id>`

SHOULD:
- Review CloudTrail for CreateFarm errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateFarm --max-results 10`
- Verify service-linked role exists for Deadline Cloud

MAY:
- Check service quotas for maximum farms per account
- Review KMS key policy for Deadline Cloud service principal access

## Phase 2 — Remediate

MUST:
- Fix IAM permissions to include deadline:CreateFarm and related actions
- Ensure KMS key policy allows deadline.amazonaws.com service principal
- Create farm: `aws deadline create-farm --display-name <name>`

SHOULD:
- Configure KMS encryption for production farms
- Tag farms for cost allocation and organization

MAY:
- Use CloudFormation or Terraform for farm creation
- Set up farm-level logging configuration

## Safety Ratings

```
safety_ratings:
  - "list-farms: GREEN — read-only farm enumeration"
  - "create-farm: YELLOW — creates top-level resource container"
  - "delete-farm: RED — destroys farm and all contained resources permanently"
  - "kms describe-key: GREEN — read-only key metadata"
```

## Escalation Conditions

- Farm creation fails with internal service errors after fixing permissions
- KMS key policy changes require security team approval
- Service quota for farms reached requiring limit increase
- Farm creation needed urgently for production render pipeline

## Data Sensitivity

- Farm metadata: LOW — farm IDs and display names
- KMS key ARN: MEDIUM — encryption key identifier
- CloudTrail events: MEDIUM — reveals IAM principals and actions
- Farm configuration: LOW — basic farm settings

## Prohibited Actions

- NEVER delete a farm with active fleets, queues, or jobs
- NEVER use a KMS key without verifying Deadline Cloud service principal access
- NEVER create farms without proper tagging for cost tracking
- NEVER share KMS key ARNs outside the operations team

## Phase 3 — Rollback

1. If farm was created with wrong configuration, update it: `aws deadline update-farm --farm-id <id>`
2. If farm must be deleted, first remove all fleets, queues, and jobs
3. If KMS key was modified, restore previous key policy
4. Document the farm creation and notify the render operations team

## Common Issues

- symptoms: "CreateFarm returns AccessDeniedException"
  diagnosis: "IAM policy missing deadline:CreateFarm permission."
  resolution: "Add deadline:CreateFarm to IAM policy. Verify no SCPs blocking Deadline Cloud actions."

- symptoms: "CreateFarm fails with KMS key error"
  diagnosis: "KMS key policy does not allow Deadline Cloud service principal."
  resolution: "Update KMS key policy to allow deadline.amazonaws.com for Encrypt, Decrypt, GenerateDataKey."

## Output Format

```yaml
root_cause: "farm_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix permissions or KMS configuration"
  long_term: "Use IaC for farm provisioning"
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
