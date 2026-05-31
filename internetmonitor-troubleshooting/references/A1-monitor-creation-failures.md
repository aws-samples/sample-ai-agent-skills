---
title: "A1 — Monitor Creation Failures"
description: "Diagnose Internet Monitor creation failures"
status: active
severity: HIGH
triggers: ["cannot create monitor", "monitor creation failed", "CreateMonitor error"]
owner: devops-agent
objective: "Resolve monitor creation issues for CloudWatch Internet Monitor"
context: "Monitors require at least one resource (CloudFront distribution or VPC) and appropriate IAM permissions."
---

## Phase 1 — Triage

MUST:
- Check existing monitors: `aws internetmonitor list-monitors`
- Verify IAM permissions for internetmonitor:CreateMonitor
- Confirm at least one valid resource ARN (CloudFront distribution or VPC) is provided

SHOULD:
- Check service quotas for maximum monitors per account: `aws service-quotas get-service-quota --service-code internetmonitor --quota-code L-4F5A0AB5`
- Verify the resource ARN format is correct (arn:aws:cloudfront::account:distribution/id or arn:aws:ec2:region:account:vpc/id)

MAY:
- Review CloudTrail for CreateMonitor errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=internetmonitor.amazonaws.com --max-results 10`

## Phase 2 — Remediate

MUST:
- Fix IAM permissions to include internetmonitor:CreateMonitor and related actions
- Provide valid resource ARNs for CloudFront distributions or VPCs
- Ensure monitor name is unique within the account and region

SHOULD:
- Set appropriate MaxCityNetworksToMonitor value based on traffic volume and budget
- Configure health event thresholds during creation

MAY:
- Enable log delivery to S3 or CloudWatch Logs during creation
- Tag the monitor for cost allocation

## Common Issues

- symptoms: "CreateMonitor returns ValidationException with invalid resource ARN"
  diagnosis: "Resource ARN format is incorrect or resource does not exist."
  resolution: "Verify resource exists: `aws cloudfront get-distribution --id <id>` or `aws ec2 describe-vpcs --vpc-ids <id>`. Use correct ARN format."

## Safety Ratings

```
safety_ratings:
  - "list-monitors: GREEN — read-only enumeration of existing monitors"
  - "CreateMonitor: YELLOW — creates a new monitor resource, incurs cost based on city-networks"
  - "service-quotas get-service-quota: GREEN — read-only quota check"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- Monitor creation fails with internal service errors after fixing all input parameters
- Service quota limit reached and quota increase request is needed
- Resource ARN is valid but monitor creation still fails with access denied
- Multiple monitors needed across regions for a coordinated deployment

## Data Sensitivity

- `aws internetmonitor list-monitors`: LOW — returns monitor metadata only
- `aws cloudfront get-distribution`: MEDIUM — reveals distribution configuration and origins
- `aws ec2 describe-vpcs`: LOW — returns VPC metadata
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principal identities and actions

## Prohibited Actions

- NEVER delete an existing monitor to work around creation limits without explicit approval
- NEVER set MaxCityNetworksToMonitor to the maximum value without discussing cost implications
- NEVER create monitors with overly broad resource lists without understanding cost impact
- NEVER ignore ValidationException details — they contain specific guidance on what is wrong

## Phase 3 — Rollback

1. If a monitor was created with incorrect configuration, update it: `aws internetmonitor update-monitor --monitor-name <name> --resources-to-remove <arn>`
2. If the monitor should not exist, delete it: `aws internetmonitor delete-monitor --monitor-name <name>` (only if confirmed not needed)
3. If IAM policy changes were made, revert to the previous policy version
4. Verify no other monitors were affected by IAM or quota changes
5. Document the creation attempt and outcome for the operations team

## Output Format

```yaml
root_cause: "monitor_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix creation parameters"
  long_term: "Deploy monitors via IaC with validated configurations"
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
