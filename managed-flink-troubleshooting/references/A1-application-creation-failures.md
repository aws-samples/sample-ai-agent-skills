---
title: "A1 — Application Creation Failures"
description: "Diagnose Managed Flink application creation failures"
status: active
severity: HIGH
triggers: ["cannot create application", "CreateApplication failed", "application creation error"]
owner: devops-agent
objective: "Resolve application creation issues including IAM, S3 code location, and runtime configuration"
context: "Applications require valid S3 code location, IAM role, runtime environment, and service-linked role."
---

## Phase 1 — Triage

MUST:
- Check existing applications: `aws kinesisanalyticsv2 list-applications`
- Verify IAM permissions for kinesisanalyticsv2:CreateApplication
- Verify S3 code location exists: `aws s3 ls s3://<bucket>/<key>`

SHOULD:
- Check service quotas: `aws service-quotas get-service-quota --service-code kinesisanalytics --quota-code L-4BCA9B87`
- Verify the application IAM role has required permissions

MAY:
- Review CloudTrail for CreateApplication errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateApplication --max-results 10`

## Phase 2 — Remediate

MUST:
- Fix IAM permissions for kinesisanalyticsv2:CreateApplication
- Ensure S3 bucket and code artifact exist in the same region
- Select a valid runtime environment (FLINK-1_15, FLINK-1_18, FLINK-1_19)

SHOULD:
- Verify application IAM role trusts kinesisanalytics.amazonaws.com
- Ensure code artifact is a valid JAR or ZIP file

MAY:
- Use CloudFormation or Terraform for repeatable application creation

## Safety Ratings

```
safety_ratings:
  - "list-applications: GREEN — read-only enumeration"
  - "s3 ls: GREEN — read-only S3 listing"
  - "CreateApplication: YELLOW — creates a new application resource"
  - "service-quotas get-service-quota: GREEN — read-only quota check"
```

## Escalation Conditions

- Application creation fails with internal service errors after fixing IAM and S3
- Service quota for applications has been reached
- Required Flink runtime version is not available in the region
- Application creation blocked by SCP or permission boundary

## Data Sensitivity

- `aws kinesisanalyticsv2 list-applications`: LOW — returns application metadata
- `aws s3 ls`: LOW — reveals S3 object listing
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER create applications with overly broad IAM roles as a workaround
- NEVER use production Kinesis streams for testing new applications
- NEVER skip runtime version validation — incompatible code will fail at start
- NEVER create applications without configuring CloudWatch logging

## Phase 3 — Rollback

1. If an application was created with incorrect configuration, delete it: `aws kinesisanalyticsv2 delete-application --application-name <name> --create-timestamp <timestamp>`
2. If an IAM role was created for testing, remove it after cleanup
3. Verify no KPU charges are incurred for the deleted application
4. Document the correct configuration for retry
5. Clean up S3 code artifacts if they were uploaded for testing

## Common Issues

- symptoms: "CreateApplication returns InvalidArgumentException for S3 location"
  diagnosis: "S3 bucket or key does not exist, or is in a different region."
  resolution: "Verify S3 bucket and key exist in the same region as the application. Check bucket permissions."

## Output Format

```yaml
root_cause: "application_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix creation issue"
  long_term: "Use IaC for deployment"
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
