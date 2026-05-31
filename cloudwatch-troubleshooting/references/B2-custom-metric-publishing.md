---
title: "B2 — Custom Metric Publishing"
description: "Diagnose PutMetricData failures for custom metrics"
status: active
severity: HIGH
triggers:
  - "PutMetricData"
  - "custom metric"
  - "publish metric"
  - "metric not publishing"
  - "high resolution metric"
owner: devops-agent
objective: "Identify why custom metrics are not being published to CloudWatch"
context: "Custom metrics are published via PutMetricData API. Common failures include IAM permission issues, invalid parameters (too many dimensions, invalid values), throttling (150 TPS per account per region for PutMetricData), and network connectivity issues from VPCs."
---

## Phase 1 — Triage

MUST:
- Verify IAM permissions include `cloudwatch:PutMetricData`
- Check for API errors in application logs or CloudTrail
- Verify metric dimensions (max 30 per metric)
- Check metric values are valid (no NaN, no Inf, timestamps within 2 weeks past or 2 hours future)

SHOULD:
- Check PutMetricData throttling: look for ThrottlingException in logs
- Verify VPC endpoint exists if publishing from a private subnet: `aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.<region>.monitoring`
- Check if using embedded metric format (EMF) vs direct API calls

MAY:
- Test with a simple PutMetricData call to isolate the issue
- Check CloudWatch Agent configuration if using the agent to publish custom metrics

## Phase 2 — Remediate

MUST:
- Fix IAM permissions to include cloudwatch:PutMetricData
- Fix invalid metric parameters (dimensions, values, timestamps)
- Add VPC endpoint for monitoring if in a private subnet

SHOULD:
- Batch metric data to reduce API calls (up to 1000 values per PutMetricData call)
- Use embedded metric format (EMF) for high-volume metric publishing from Lambda
- Implement exponential backoff for throttling

MAY:
- Use high-resolution (StorageResolution=1) only when sub-minute granularity is needed
- Monitor PutMetricData API call count via CloudTrail

## Common Issues

- symptoms: "ThrottlingException on PutMetricData"
  diagnosis: "Exceeding 150 TPS limit for PutMetricData per account per region."
  resolution: "Batch metrics (up to 1000 values per call), implement backoff, or request limit increase."

- symptoms: "InvalidParameterValue on PutMetricData"
  diagnosis: "Metric value is NaN/Inf, timestamp out of range, or more than 30 dimensions."
  resolution: "Validate metric values, ensure timestamps are within 2 weeks past to 2 hours future, limit to 30 dimensions."

- symptoms: "Custom metric published but not visible"
  diagnosis: "Metric published to wrong region or with unexpected namespace/dimensions."
  resolution: "Check the region in the API call. Use list-metrics to find the actual namespace and dimensions."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix IAM permissions for PutMetricData | YELLOW | IAM changes have broad impact potential |
| Fix metric parameters | GREEN | Application-side fix |
| Add VPC endpoint for monitoring | GREEN | Additive network configuration |
| Implement batching and backoff | GREEN | Application-side optimization |

## Escalation Conditions

- Custom metrics not publishing for production monitoring — escalate to application team
- PutMetricData throttling affecting monitoring — escalate to operations team for limit increase
- VPC endpoint creation in production VPCs — escalate to network team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| Application logs (PutMetricData errors) | MEDIUM | May contain metric names and values |
| `aws ec2 describe-vpc-endpoints` | LOW | Network configuration |

## Prohibited Actions

- **NEVER** suggest deleting log groups with retention data without confirming data is no longer needed
- **NEVER** suggest disabling alarms in production
- **NEVER** suggest removing metric filters that feed production alarms
- **NEVER** publish metrics with sensitive data in dimension names or values

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Fixed IAM permissions | Revert IAM policy changes |
| Added VPC endpoint | Delete: `aws ec2 delete-vpc-endpoints` |
| Application-side fixes | Revert application code changes |

## Output Format

```yaml
root_cause: "custom_metric_publishing — <specific_cause>"
evidence:
  - type: api_errors
    content: "<PutMetricData error details>"
  - type: iam_permissions
    content: "<IAM policy for cloudwatch:PutMetricData>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions, parameters, or network connectivity"
  long_term: "Use EMF for high-volume publishing, implement batching and backoff"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
