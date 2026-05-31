---
title: "A1 — Detector Creation Failures"
description: "Diagnose and resolve Amazon Lookout for Metrics detector creation failures"
status: active
severity: HIGH
triggers: ["cannot create detector", "detector creation failed", "CreateAnomalyDetector error"]
owner: devops-agent
objective: "Resolve detector creation issues and establish a working anomaly detector"
context: "Detectors require valid configuration including frequency, metric sets, and IAM permissions. Creation failures are typically caused by IAM issues, quota limits, or invalid parameters."
---

## Phase 1 — Triage

MUST:
- List existing detectors to check for duplicates: `aws lookoutmetrics list-anomaly-detectors`
- Verify IAM permissions for `lookoutmetrics:CreateAnomalyDetector` and `lookoutmetrics:CreateMetricSet`
- Check service quotas: `aws service-quotas get-service-quota --service-code lookoutmetrics --quota-code L-1234ABCD`

SHOULD:
- Review CloudTrail for CreateAnomalyDetector error details: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateAnomalyDetector --max-results 10`
- Verify the detector frequency (P1D, PT1H, PT10M, PT5M) matches data availability

MAY:
- Check if the region supports Lookout for Metrics
- Review IAM role trust policy for lookoutmetrics.amazonaws.com service principal

## Phase 2 — Remediate

MUST:
- Fix IAM permissions — ensure the execution role has `lookoutmetrics:*` and data source access (S3, CloudWatch, etc.)
- Correct invalid parameters (frequency, detector name format, description length)
- Ensure service quota for number of detectors is not exceeded

SHOULD:
- Create the IAM service-linked role if missing: `aws iam create-service-linked-role --aws-service-name lookoutmetrics.amazonaws.com`
- Validate the detector configuration JSON before submission

MAY:
- Request quota increase if detector limit is reached
- Use CloudFormation or Terraform for repeatable detector creation

## Safety Ratings

```
safety_ratings:
  - "list-anomaly-detectors: GREEN — read-only enumeration"
  - "CreateAnomalyDetector: YELLOW — creates a new detector resource, incurs cost"
  - "create-service-linked-role: YELLOW — creates IAM role, non-destructive"
  - "service-quotas get-service-quota: GREEN — read-only quota check"
```

## Escalation Conditions

- Detector creation fails with InternalServerException after all configuration fixes
- Service quota increase request is denied
- IAM permissions are correct but creation still fails with AccessDeniedException (possible SCP)
- Region availability issues for Lookout for Metrics

## Data Sensitivity

- `aws lookoutmetrics list-anomaly-detectors`: LOW — returns detector metadata only
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principal identities and actions
- `aws iam create-service-linked-role`: LOW — creates a predefined service role
- Detector configuration JSON: MEDIUM — may contain data source connection details

## Prohibited Actions

- NEVER delete an existing active detector to resolve creation issues for a new one
- NEVER create detectors with overly broad IAM permissions (e.g., `*` on all resources)
- NEVER bypass SCP restrictions without approval from the organization administrator
- NEVER share detector ARNs or configuration details in public channels

## Phase 3 — Rollback

1. If a detector was created with incorrect configuration, delete it: `aws lookoutmetrics delete-anomaly-detector --anomaly-detector-arn <arn>` (only if not yet activated or confirmed non-production)
2. If a service-linked role was created unnecessarily, it will be cleaned up automatically when unused
3. If quota increase was requested in error, contact AWS Support to cancel
4. Document the creation attempt and any configuration changes made
5. Verify no other detectors were affected by IAM policy changes

## Common Issues

- symptoms: "CreateAnomalyDetector returns ValidationException"
  diagnosis: "Invalid parameter values — check frequency, name format, or description length."
  resolution: "Correct parameters. Name must be 1-63 characters, alphanumeric and hyphens only."

- symptoms: "AccessDeniedException despite correct IAM policy"
  diagnosis: "SCP or permissions boundary blocking lookoutmetrics actions."
  resolution: "Check SCPs: `aws organizations list-policies --filter SERVICE_CONTROL_POLICY` and review for lookoutmetrics deny rules."

## Output Format

```yaml
root_cause: "detector_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details from CloudTrail or API response>"
severity: HIGH
mitigation:
  immediate: "Fix creation parameters or IAM permissions"
  long_term: "Implement IaC templates for consistent detector creation"
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
