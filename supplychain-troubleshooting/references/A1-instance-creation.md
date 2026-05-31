---
title: "A1 — Instance Creation"
description: "Diagnose Supply Chain instance creation failures"
status: active
severity: HIGH
triggers: ["cannot create instance", "instance creation failed", "CreateInstance error"]
owner: devops-agent
objective: "Resolve Supply Chain instance creation issues"
context: "Instances are the top-level resource. Creation is asynchronous and can take several minutes."
---

## Phase 1 — Triage

MUST:
- Check existing instances: `aws scn list-instances`
- Verify IAM permissions for scn:CreateInstance
- Check service quotas: `aws service-quotas list-service-quotas --service-code scn`

SHOULD:
- Verify the service is available in the target region
- Check if account has reached instance limit

MAY:
- Review CloudTrail for CreateInstance errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=scn.amazonaws.com --max-results 10`

## Phase 2 — Remediate

MUST:
- Fix IAM permissions: ensure scn:CreateInstance is allowed
- Request quota increase if instance limit reached
- Verify region availability for Supply Chain

SHOULD:
- Use descriptive instance names for identification
- Tag instances for cost allocation

MAY:
- Automate instance creation with CloudFormation

## Safety Ratings

```
safety_ratings:
  - "list-instances: GREEN — read-only enumeration of existing instances"
  - "CreateInstance: YELLOW — creates a new instance with associated costs"
  - "service-quotas list-service-quotas: GREEN — read-only quota check"
  - "CloudFormation deployment: YELLOW — creates infrastructure resources"
```

## Escalation Conditions

- Instance creation fails with internal service errors after IAM fixes
- Service quota increase request is denied
- Instance stuck in INITIALIZING state for extended period
- Multiple instances needed across regions for compliance

## Data Sensitivity

- `aws scn list-instances`: LOW — returns instance metadata only
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principal identities
- Instance configuration: LOW — contains operational metadata

## Prohibited Actions

- NEVER create instances in unsupported regions
- NEVER delete existing instances to work around quota limits
- NEVER share instance credentials or access tokens
- NEVER create instances without confirming cost implications

## Phase 3 — Rollback

1. If an instance was created in error: delete it: `aws scn delete-instance --instance-id <id>`
2. If CloudFormation stack was deployed incorrectly: delete the stack
3. If quota increase was requested in error: contact AWS Support to cancel
4. Verify no data was ingested into incorrectly configured instances
5. Document the rollback action and notify the operations team

## Common Issues

- symptoms: "CreateInstance returns AccessDeniedException"
  diagnosis: "IAM policy missing scn:CreateInstance permission."
  resolution: "Add scn:CreateInstance to the IAM policy."

- symptoms: "Instance stuck in INITIALIZING state"
  diagnosis: "Instance provisioning is still in progress."
  resolution: "Wait up to 15 minutes. If still initializing, check CloudTrail for errors and contact support."

## Output Format

```yaml
root_cause: "instance_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix creation issue"
  long_term: "Automate instance provisioning via IaC"
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
