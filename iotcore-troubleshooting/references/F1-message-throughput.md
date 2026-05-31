---
title: "F1 — Message Throughput Issues"
description: "Diagnose message throughput limitations and optimization"
status: active
severity: MEDIUM
triggers:
  - "message throughput low"
  - "publish rate limited"
  - "messages delayed"
  - "throughput bottleneck"
  - "message delivery slow"
owner: devops-agent
objective: "Identify and resolve message throughput issues"
context: "Throughput issues include: per-account publish limits, per-connection limits, message size limits (128 KB), or inefficient message patterns."
---

## Phase 1 — Triage

MUST:
- Check publish metrics: `aws cloudwatch get-metric-statistics --namespace AWS/IoT --metric-name PublishIn.Success --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check throttling: `aws cloudwatch get-metric-statistics --namespace AWS/IoT --metric-name PublishIn.Throttle --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check service quotas: `aws service-quotas get-service-quota --service-code iot --quota-code L-66D47A63`
- Verify message sizes are within 128 KB limit

SHOULD:
- Check per-connection publish rate
- Review message patterns for optimization opportunities
- Check rule engine processing throughput

MAY:
- Analyze message size distribution
- Check for fan-out patterns causing amplification

## Phase 2 — Remediate

MUST:
- Request quota increase if hitting limits: `aws service-quotas request-service-quota-increase --service-code iot --quota-code L-66D47A63 --desired-value <value>`
- Reduce message size if over 128 KB
- Optimize publish patterns

SHOULD:
- Batch small messages to reduce publish rate
- Use MQTT 5 features for efficiency where supported
- Implement client-side throttling

MAY:
- Use Basic Ingest ($aws/rules/<rule-name>) to bypass message broker
- Implement message compression

## Output Format

```yaml
root_cause: "throughput_issue — <specific_cause>"
evidence:
  - type: publish_metrics
    content: "<publish rate and throttling>"
  - type: quotas
    content: "<current limits>"
severity: MEDIUM
mitigation:
  immediate: "Request quota increase or optimize message patterns"
  long_term: "Implement efficient messaging architecture"
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
