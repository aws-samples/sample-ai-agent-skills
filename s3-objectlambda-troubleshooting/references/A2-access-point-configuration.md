---
title: "A2 — Access Point Configuration"
description: "Diagnose Object Lambda access point configuration issues"
status: active
severity: MEDIUM
triggers: ["OLAP misconfigured", "wrong transformation", "access point configuration error"]
owner: devops-agent
objective: "Resolve Object Lambda access point configuration problems"
context: "OLAP configuration includes transformation actions, Lambda ARN, and allowed features. Misconfiguration causes silent failures."
---

## Phase 1 — Triage

MUST:
- Get current OLAP configuration: `aws s3control get-access-point-configuration-for-object-lambda --account-id <account-id> --name <olap-name>`
- Verify Lambda function ARN is correct: `aws lambda get-function --function-name <function-arn>`
- Check which actions are configured (GetObject, HeadObject, ListObjects)

SHOULD:
- Verify AllowedFeatures includes required features (GetObject-Range, GetObject-PartNumber, HeadObject-Range, HeadObject-PartNumber)
- Check if ContentTransformation specifies the correct Lambda ARN

MAY:
- Compare configuration against a known-good OLAP
- Review CloudWatch metrics for the OLAP to identify error patterns

## Phase 2 — Remediate

MUST:
- Update OLAP configuration with correct Lambda ARN: `aws s3control put-access-point-configuration-for-object-lambda --account-id <account-id> --name <olap-name> --configuration file://config.json`
- Enable required AllowedFeatures for range/part-number support
- Ensure at least one transformation action is configured

SHOULD:
- Test configuration changes with a non-production OLAP first
- Validate the Lambda function handles all configured action types

MAY:
- Enable CloudWatch request metrics for the OLAP
- Document the configuration in a runbook for the team

## Common Issues

- symptoms: "Range GET requests return full object instead of requested range"
  diagnosis: "AllowedFeatures does not include GetObject-Range."
  resolution: "Update OLAP configuration to include GetObject-Range in AllowedFeatures."

- symptoms: "HEAD requests bypass Lambda transformation"
  diagnosis: "HeadObject action is not configured in the OLAP transformation configuration."
  resolution: "Add HeadObject to the TransformationConfigurations actions list."

## Safety Ratings

```
safety_ratings:
  - "get-access-point-configuration-for-object-lambda: GREEN — read-only configuration retrieval"
  - "put-access-point-configuration-for-object-lambda: YELLOW — modifies OLAP behavior"
  - "lambda get-function: GREEN — read-only function metadata"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
```

## Escalation Conditions

- Configuration changes do not take effect after 5 minutes
- OLAP returns 500 errors after configuration update
- Multiple transformation actions conflict with each other
- Configuration requires features not available in the current region

## Data Sensitivity

- `aws s3control get-access-point-configuration-for-object-lambda`: LOW — returns configuration metadata
- `aws lambda get-function`: MEDIUM — may reveal environment variables
- `aws cloudwatch get-metric-statistics`: LOW — returns aggregated metrics
- `aws s3control put-access-point-configuration-for-object-lambda`: MEDIUM — modifies access behavior

## Prohibited Actions

- NEVER remove transformation actions from a production OLAP without verifying no clients depend on them
- NEVER change the Lambda function ARN in production without testing the new function
- NEVER disable AllowedFeatures that active clients rely on
- NEVER apply configuration changes during peak traffic without a maintenance window

## Phase 3 — Rollback

1. Restore previous OLAP configuration: `aws s3control put-access-point-configuration-for-object-lambda --account-id <account-id> --name <olap-name> --configuration file://previous-config.json`
2. Verify the rollback by testing a GET request through the OLAP
3. Check CloudWatch metrics to confirm error rates return to baseline
4. Notify dependent teams of the configuration rollback
5. Document the failed configuration change and root cause

## Output Format

```yaml
root_cause: "olap_configuration — <specific_cause>"
evidence:
  - type: configuration_error
    content: "<configuration details>"
severity: MEDIUM
mitigation:
  immediate: "Fix OLAP configuration"
  long_term: "Version-control OLAP configurations in IaC"
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
  - command: "get-bucket-policy"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-bucket-acl"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-public-access-block"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling S3 Block Public Access"
  - "NEVER suggest Principal: * without Condition keys"
  - "NEVER suggest removing bucket encryption"
