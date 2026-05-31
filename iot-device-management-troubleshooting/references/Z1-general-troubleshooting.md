---
title: "Z1 — General Troubleshooting"
description: "Catch-all runbook for IoT Device Management issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["IoT device management error", "unknown IoT issue", "general IoT problem", "IoT not working"]
owner: devops-agent
objective: "Systematically diagnose and resolve general IoT Device Management issues"
context: "This runbook covers general troubleshooting when the specific issue category is unclear. Start here and route to specific runbooks as the problem domain becomes clear."
---

## Phase 1 — Triage

MUST:
- Check IoT endpoint: `aws iot describe-endpoint --endpoint-type iot:Data-ATS`
- List things to verify basic connectivity: `aws iot list-things --max-results 10`
- Check IoT logging configuration: `aws iot get-v2-logging-options`

SHOULD:
- Review recent CloudTrail events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=iot.amazonaws.com --max-results 20`
- Check service quotas: `aws service-quotas list-service-quotas --service-code iot --query 'Quotas[?UsageMetric].{Name:QuotaName,Value:Value}'`
- Verify IAM permissions for the calling principal

MAY:
- Check CloudWatch metrics for IoT: `aws cloudwatch list-metrics --namespace AWS/IoT --query 'Metrics[*].MetricName' --output text | tr '\t' '\n' | sort -u`
- Review IoT Core logs: `aws logs filter-log-events --log-group-name AWSIotLogsV2 --start-time <epoch-ms> --filter-pattern "ERROR"`
- Check for service health events in the region

## Phase 2 — Remediate

MUST:
- Enable IoT logging if not configured: `aws iot set-v2-logging-options --role-arn <log-role-arn> --default-log-level INFO`
- Verify the IoT data endpoint is correct and reachable from devices
- Check IAM and IoT policies allow the required operations

SHOULD:
- Route to specific runbook once issue domain is identified (A1-F1)
- Check if the issue is region-specific by testing in another region
- Verify AWS SDK and CLI versions are current

MAY:
- Enable DEBUG logging temporarily for detailed diagnostics
- Create a test thing to validate basic IoT operations end-to-end
- Check AWS Health Dashboard for IoT service issues

## Safety Ratings

```
safety_ratings:
  - "describe-endpoint: GREEN — read-only endpoint retrieval"
  - "list-things: GREEN — read-only enumeration"
  - "get-v2-logging-options: GREEN — read-only logging config"
  - "set-v2-logging-options: YELLOW — changes logging level, may increase costs"
  - "cloudtrail lookup-events: GREEN — read-only audit log query"
```

## Escalation Conditions

- Multiple IoT services failing simultaneously indicating regional issue
- Service quota limits reached blocking device operations
- Persistent 5xx errors from IoT APIs suggesting service degradation
- Security incident involving compromised device certificates or credentials

## Data Sensitivity

- `aws iot describe-endpoint`: LOW — returns endpoint URL
- `aws iot list-things`: MEDIUM — returns device identifiers
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principals
- `aws logs filter-log-events`: MEDIUM — may contain device identifiers and error details

## Prohibited Actions

- NEVER delete things, certificates, or policies during general troubleshooting
- NEVER modify production IoT policies without understanding the impact
- NEVER disable IoT logging during an active investigation
- NEVER share IoT endpoint URLs or device certificates in unsecured channels

## Phase 3 — Rollback

1. If logging level was changed, restore: `aws iot set-v2-logging-options --role-arn <arn> --default-log-level INFO`
2. If test resources were created, clean them up after investigation
3. Document findings and route to appropriate specific runbook
4. Restore any temporarily modified configurations
5. Notify the team of findings and recommended next steps

## Common Issues

- symptoms: "All IoT API calls return UnauthorizedException"
  diagnosis: "IAM credentials expired, insufficient permissions, or STS token issue."
  resolution: "Verify IAM credentials. Check IAM policy allows iot:* actions. Refresh STS credentials if using assumed roles."

- symptoms: "Devices cannot connect to IoT endpoint"
  diagnosis: "Incorrect endpoint URL, expired device certificate, or network connectivity issue."
  resolution: "Verify endpoint with describe-endpoint. Check certificate status. Test network connectivity to endpoint on port 8883."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: general_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Route to specific runbook for comprehensive resolution"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
