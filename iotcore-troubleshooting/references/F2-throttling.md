---
title: "F2 — Throttling Issues"
description: "Diagnose IoT Core API and message throttling"
status: active
severity: MEDIUM
triggers:
  - "throttling"
  - "rate exceeded"
  - "503 error"
  - "too many requests"
  - "quota exceeded"
owner: devops-agent
objective: "Identify and resolve throttling issues"
context: "Throttling occurs when: publish/subscribe rate exceeds limits, API call rate exceeded, rule engine throughput exceeded, or shadow update rate too high."
---

## Phase 1 — Triage

MUST:
- Check throttling metrics: `aws cloudwatch get-metric-statistics --namespace AWS/IoT --metric-name PublishIn.Throttle --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check subscribe throttling: `aws cloudwatch get-metric-statistics --namespace AWS/IoT --metric-name Subscribe.Throttle --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check rule throttling: `aws cloudwatch get-metric-statistics --namespace AWS/IoT --metric-name RuleMessageThrottled --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Review current service quotas: `aws service-quotas list-service-quotas --service-code iot`

SHOULD:
- Identify which specific limit is being hit
- Check shadow update rate if shadow throttling
- Review API call patterns for management API throttling

MAY:
- Analyze throttling patterns over time
- Check for burst vs sustained throttling

## Phase 2 — Remediate

MUST:
- Request quota increases for specific limits
- Implement client-side rate limiting and backoff
- Distribute load across time if possible

SHOULD:
- Use exponential backoff with jitter for retries
- Use Basic Ingest to bypass message broker for rule-only messages
- Batch operations where possible

MAY:
- Implement message queuing on device side
- Use multiple AWS accounts for very high throughput

## Output Format

```yaml
root_cause: "throttling — <specific_cause>"
evidence:
  - type: throttle_metrics
    content: "<throttling counts by type>"
  - type: quotas
    content: "<current vs needed limits>"
severity: MEDIUM
mitigation:
  immediate: "Request quota increase and implement backoff"
  long_term: "Design for rate limits with client-side throttling"
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
