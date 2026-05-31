---
title: "A2 — Detector Activation Errors"
description: "Diagnose and resolve detector activation failures"
status: active
severity: HIGH
triggers: ["detector won't activate", "activation failed", "ActivateAnomalyDetector error", "detector stuck in INACTIVE"]
owner: devops-agent
objective: "Successfully activate a detector and begin the learning period"
context: "Activation validates the entire configuration including data source connectivity, metric set schema, and IAM permissions. Failures at activation indicate configuration issues that passed initial creation validation."
---

## Phase 1 — Triage

MUST:
- Check detector status: `aws lookoutmetrics describe-anomaly-detector --anomaly-detector-arn <arn>`
- Verify metric set configuration: `aws lookoutmetrics describe-metric-set --metric-set-arn <arn>`
- Test data source connectivity by verifying data exists at the configured location

SHOULD:
- Check data quality metrics if detector was previously active: `aws lookoutmetrics get-data-quality-metrics --anomaly-detector-arn <arn>`
- Review CloudTrail for ActivateAnomalyDetector errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=ActivateAnomalyDetector --max-results 10`

MAY:
- Verify the IAM role can assume the lookoutmetrics service principal
- Check if the data source has sufficient historical data for the configured frequency

## Phase 2 — Remediate

MUST:
- Fix data source connectivity issues (S3 bucket access, CloudWatch permissions, database credentials)
- Correct metric set schema mismatches (timestamp format, measure types, dimension names)
- Ensure the IAM execution role has access to all configured data sources

SHOULD:
- Validate S3 path template matches actual folder structure
- Verify CloudWatch metrics exist with exact namespace/name/dimensions
- Test database connectivity from the configured VPC/subnet

MAY:
- Pre-populate S3 with historical data to accelerate the learning period
- Create a test detector with minimal configuration to isolate the issue

## Safety Ratings

```
safety_ratings:
  - "describe-anomaly-detector: GREEN — read-only status check"
  - "ActivateAnomalyDetector: YELLOW — starts the detector and begins incurring costs"
  - "DeactivateAnomalyDetector: YELLOW — stops detection, resets learning period on reactivation"
  - "describe-metric-set: GREEN — read-only configuration check"
```

## Escalation Conditions

- Activation fails with internal service errors after all configuration fixes
- Data source connectivity cannot be established despite correct network configuration
- Detector enters FAILED status with no actionable error message
- Activation succeeds but detector transitions to BACK_TEST_FAILED status

## Data Sensitivity

- `aws lookoutmetrics describe-anomaly-detector`: LOW — returns detector configuration
- `aws lookoutmetrics describe-metric-set`: MEDIUM — may contain data source connection details
- `aws lookoutmetrics get-data-quality-metrics`: LOW — returns quality statistics only
- Data source credentials in metric set configuration: HIGH — contains connection strings

## Prohibited Actions

- NEVER deactivate a production detector to test activation of another detector
- NEVER modify metric set configuration on an active detector without understanding the learning period reset
- NEVER expose database credentials from metric set configuration in logs or outputs
- NEVER activate a detector without confirming the data source has data available

## Phase 3 — Rollback

1. If activation was triggered in error, deactivate immediately: `aws lookoutmetrics deactivate-anomaly-detector --anomaly-detector-arn <arn>`
2. If metric set was modified incorrectly, update it back to the original configuration before reactivating
3. If IAM role permissions were broadened for testing, revert to least-privilege after resolution
4. Document the activation attempt and any configuration changes
5. Notify stakeholders if the learning period was reset due to deactivation

## Common Issues

- symptoms: "Detector status shows FAILED after activation"
  diagnosis: "Data source is unreachable or returns no data for the configured time window."
  resolution: "Verify data source connectivity. For S3, check bucket policy and path template. For CloudWatch, verify metrics exist."

- symptoms: "Activation succeeds but no anomalies after 2+ weeks"
  diagnosis: "Data quality issues — insufficient data points or too many missing values."
  resolution: "Check data quality: `aws lookoutmetrics get-data-quality-metrics --anomaly-detector-arn <arn>` and fix data pipeline."

## Output Format

```yaml
root_cause: "detector_activation — <specific_cause>"
evidence:
  - type: activation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix activation blocker"
  long_term: "Implement pre-activation validation checks"
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
