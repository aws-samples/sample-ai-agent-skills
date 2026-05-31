---
title: "Z1 — General Troubleshooting"
description: "Catch-all runbook for IoT FleetWise issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["FleetWise error", "unknown FleetWise issue", "general FleetWise problem", "FleetWise not working"]
owner: devops-agent
objective: "Systematically diagnose and resolve general IoT FleetWise issues"
context: "This runbook covers general troubleshooting when the specific issue category is unclear. Start here and route to specific runbooks as the problem domain becomes clear."
---

## Phase 1 — Triage

MUST:
- Check FleetWise logging: `aws iotfleetwise get-logging-options`
- List core resources: `aws iotfleetwise list-signal-catalogs` and `aws iotfleetwise list-fleets`
- Verify IAM caller permissions: `aws sts get-caller-identity`

SHOULD:
- Review recent CloudTrail events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=iotfleetwise.amazonaws.com --max-results 20`
- Check service quotas: `aws service-quotas list-service-quotas --service-code iotfleetwise`
- Verify FleetWise is available in the current region

MAY:
- Check CloudWatch metrics for FleetWise
- Review AWS Health Dashboard for service issues
- List all resources to assess overall configuration state

## Phase 2 — Remediate

MUST:
- Enable FleetWise logging if not configured: `aws iotfleetwise put-logging-options --cloud-watch-log-delivery '{logType: "ERROR", logGroupName: "/aws/iotfleetwise"}'`
- Verify the data model chain: signal catalog → model manifest → decoder manifest → vehicle → fleet → campaign
- Check IAM permissions for the calling principal

SHOULD:
- Route to specific runbook once issue domain is identified (A1-F1)
- Check if the issue is region-specific
- Verify AWS CLI and SDK versions support FleetWise

MAY:
- Create test resources to validate basic FleetWise operations
- Enable DEBUG-level logging temporarily
- Check AWS documentation for recent service changes or known issues

## Safety Ratings

```
safety_ratings:
  - "get-logging-options: GREEN — read-only logging config"
  - "list-signal-catalogs: GREEN — read-only enumeration"
  - "list-fleets: GREEN — read-only enumeration"
  - "put-logging-options: YELLOW — enables logging, may increase CloudWatch costs"
  - "cloudtrail lookup-events: GREEN — read-only audit log query"
```

## Escalation Conditions

- Multiple FleetWise services failing simultaneously
- Service quota limits reached blocking operations
- Persistent 5xx errors from FleetWise APIs
- Data loss affecting compliance-required vehicle telemetry

## Data Sensitivity

- `aws iotfleetwise get-logging-options`: LOW — returns logging configuration
- `aws iotfleetwise list-signal-catalogs`: LOW — returns catalog metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history
- FleetWise logs: MEDIUM — may contain vehicle identifiers and error details

## Prohibited Actions

- NEVER delete the signal catalog during general troubleshooting
- NEVER modify production campaigns without understanding the impact
- NEVER disable FleetWise logging during an active investigation
- NEVER share vehicle data or telemetry in unsecured channels

## Phase 3 — Rollback

1. If logging was changed, restore: `aws iotfleetwise put-logging-options --cloud-watch-log-delivery '{logType: "OFF"}'`
2. If test resources were created, clean them up after investigation
3. Document findings and route to appropriate specific runbook
4. Restore any temporarily modified configurations
5. Notify the vehicle data team of findings and next steps

## Common Issues

- symptoms: "All FleetWise API calls return UnrecognizedClientException"
  diagnosis: "AWS CLI or SDK version does not support FleetWise, or FleetWise is not available in the region."
  resolution: "Update AWS CLI to latest version. Verify FleetWise regional availability."

- symptoms: "Resources exist but data pipeline is not working end-to-end"
  diagnosis: "Break in the data model chain: signal catalog → model manifest → decoder manifest → vehicle → fleet → campaign."
  resolution: "Verify each link in the chain. Check manifest statuses are ACTIVE. Verify campaign is RUNNING. Check edge agent connectivity."

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
