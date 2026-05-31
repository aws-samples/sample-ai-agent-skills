---
title: "Z1 — General IoT Core Troubleshooting"
description: "Catch-all systematic investigation for IoT Core issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "IoT Core issue"
  - "IoT problem"
  - "IoT not working"
  - "IoT error"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate IoT Core issues using a structured diagnostic approach"
context: "This runbook provides a general-purpose investigation framework for IoT Core issues that don't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- Check IoT endpoint: `aws iot describe-endpoint --endpoint-type iot:Data-ATS`
- Check thing status: `aws iot describe-thing --thing-name <thing-name>`
- Check certificate and policy: `aws iot list-thing-principals --thing-name <thing-name>`
- Check IoT logs: `aws logs filter-log-events --log-group-name AWSIotLogsV2 --start-time <epoch> --limit 50`
- Check key metrics: Connect.Success, PublishIn.Success, Subscribe.Success

SHOULD:
- Check rule engine status: `aws iot list-topic-rules`
- Check device shadow: `aws iot-data get-thing-shadow --thing-name <thing-name> /dev/stdout`
- Review CloudTrail for recent IoT changes
- Check fleet indexing status

MAY:
- Check AWS Health Dashboard for IoT service issues
- Review IoT Device Defender audit results
- Check billing for unexpected usage patterns

## Phase 2 — Remediate

MUST:
- Based on triage findings, pivot to the appropriate specific runbook:
  - Connectivity → A1 (MQTT), A2 (certificates), A3 (policy denied)
  - Rules → B1 (engine errors), B2 (action failures), B3 (SQL)
  - Shadows → C1 (sync), C2 (conflicts)
  - Registry → D1 (registration), D2 (groups)
  - Security → E1 (cert management), E2 (policies), E3 (authorizers)
  - Performance → F1 (throughput), F2 (throttling)
  - Integration → G1 (Lambda), G2 (S3/DynamoDB/Kinesis)
- Document findings and root cause

SHOULD:
- Check for multiple concurrent issues
- Review recent configuration changes

MAY:
- Enable detailed IoT logging if not already enabled
- Set up comprehensive monitoring

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: iot_status
    content: "<endpoint and thing status>"
  - type: connectivity
    content: "<connection metrics>"
  - type: logs
    content: "<relevant log entries>"
severity: MEDIUM
general_analysis:
  connectivity: "healthy | failing"
  rules: "active | errors"
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow the identified specific runbook"
  long_term: "Implement comprehensive IoT monitoring"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never revoke or delete device certificates without confirming device decommission
- Never modify IoT policies on production devices without testing in a staging environment
- Never delete thing groups that contain active production devices
- Never alter MQTT topic rules that route critical telemetry data
- Never disable custom authorizers serving active device connections

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Reattach previous certificates if device connectivity is lost after changes
- Restore previous IoT policy versions if devices lose permissions
- Revert rule engine action configurations if message routing breaks

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
