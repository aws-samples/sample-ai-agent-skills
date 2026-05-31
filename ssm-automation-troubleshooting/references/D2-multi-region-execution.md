---
title: "D2 — Multi-Region Execution"
description: "Diagnose multi-region SSM Automation execution failures including regional document availability and execution coordination"
status: active
severity: HIGH
triggers: ["multi-region failed", "region execution error", "document not found in region", "regional automation failure"]
owner: devops-agent
objective: "Resolve multi-region automation execution failures"
context: "Multi-region automation creates separate child executions in each target region. Documents must exist in each region. Regional executions are independent — failure in one region does not stop others unless configured."
---

## Phase 1 — Triage

MUST:
- Get parent execution details: `aws ssm get-automation-execution --automation-execution-id <execution-id>`
- Check child executions across regions: `aws ssm describe-automation-executions --filters Key=ParentExecutionId,Values=<parent-id>`
- Verify document exists in target regions: `aws ssm describe-document --name <document-name> --region <target-region>`

SHOULD:
- Check TargetLocations configuration for region list
- Verify IAM role is available in each target region
- Review regional service availability for actions in the document

MAY:
- Check regional CloudTrail for execution events
- Verify VPC and networking prerequisites in each region

## Phase 2 — Remediate

MUST:
- Deploy the automation document to all target regions
- Ensure IAM roles and permissions are consistent across regions
- Fix region-specific failures (resource availability, service limits)

SHOULD:
- Use CloudFormation StackSets for multi-region document deployment
- Configure TargetLocationMaxConcurrency and TargetLocationMaxErrors
- Test in non-production regions before production deployment

MAY:
- Implement region-specific parameter overrides for regional differences
- Set up cross-region monitoring for automation execution status

## Safety Ratings

```
safety_ratings:
  - "describe-automation-executions: GREEN — read-only execution listing"
  - "describe-document with --region: GREEN — read-only document check per region"
  - "start-automation-execution with TargetLocations: RED — executes across multiple regions"
  - "create-document with --region: YELLOW — creates document in specific region"
```

## Escalation Conditions

- Multi-region execution partially completes leaving regions in inconsistent state
- Document deployment to new regions requires security review
- Regional service outage affects automation in specific regions
- Cross-region execution creates resources that incur unexpected costs

## Data Sensitivity

- `aws ssm describe-automation-executions`: LOW — returns execution metadata
- `aws ssm get-automation-execution`: MEDIUM — contains parameters and target details
- `aws ssm describe-document --region`: LOW — returns document metadata per region
- `aws cloudtrail lookup-events --region`: MEDIUM — reveals regional execution activity

## Prohibited Actions

- NEVER deploy automation documents to regions without verifying regional compliance requirements
- NEVER run multi-region automation without testing in a single region first
- NEVER ignore regional failures — investigate each independently
- NEVER assume all regions have identical service availability and limits

## Phase 3 — Rollback

1. Stop the parent execution: `aws ssm stop-automation-execution --automation-execution-id <parent-id> --type Cancel`
2. Identify which regional child executions completed, failed, or were cancelled
3. For each affected region, review resource changes and revert if needed
4. If documents were deployed to wrong regions, remove them: `aws ssm delete-document --name <name> --region <region>`
5. Ensure all regions are in a consistent state before re-attempting

## Common Issues

- symptoms: "Execution fails in some regions with DocumentNotFound"
  diagnosis: "The automation document does not exist in the target region."
  resolution: "Deploy the document to all target regions. Use StackSets for consistent deployment."

- symptoms: "Regional execution fails with service-specific errors"
  diagnosis: "The AWS service or resource type used in the document is not available in that region."
  resolution: "Check regional service availability. Exclude unsupported regions from TargetLocations."

- symptoms: "Multi-region execution is very slow"
  diagnosis: "TargetLocationMaxConcurrency is too low, executing regions sequentially."
  resolution: "Increase TargetLocationMaxConcurrency to allow parallel regional execution."

## Output Format

```yaml
root_cause: "multi_region — <specific_cause>"
evidence:
  - type: regional_error
    content: "<region-specific failure details>"
severity: HIGH
mitigation:
  immediate: "Fix regional execution failures"
  long_term: "Standardize multi-region document deployment"
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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
