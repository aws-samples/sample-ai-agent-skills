---
title: "F1 — App Mesh Access Logging Issues"
description: "Diagnose access logging configuration and delivery problems"
status: active
severity: LOW
triggers:
  - "access logs missing"
  - "logging not working"
  - "no access logs"
  - "envoy access log"
owner: devops-agent
objective: "Identify and resolve access logging configuration issues in App Mesh"
context: "App Mesh supports access logging on virtual nodes and virtual gateways. Logs can be sent to stdout/stderr (for CloudWatch via container logging) or to a file path. Access logs provide request-level visibility."
---

## Phase 1 — Triage

MUST:
- Check virtual node logging config: `aws appmesh describe-virtual-node --mesh-name <mesh-name> --virtual-node-name <node-name> --query 'virtualNode.spec.logging'`
- Check Envoy access logs: `kubectl logs <pod-name> -c envoy | grep -E "^\[" | tail -20`
- Verify logging is configured on the virtual node spec

SHOULD:
- Check log format configuration
- Verify container log driver is configured to capture Envoy stdout
- Check CloudWatch log group for Envoy logs

MAY:
- Customize access log format for specific fields
- Check Envoy stats for logged request counts

## Phase 2 — Remediate

MUST:
- Configure access logging on virtual node: set logging.accessLog.file.path to `/dev/stdout`
- Ensure container log driver captures Envoy container output
- Verify log format includes required fields

SHOULD:
- Use structured JSON log format for easier parsing
- Configure CloudWatch Logs agent or Fluent Bit for log collection
- Set appropriate log retention policies

MAY:
- Add custom access log fields for debugging
- Set up log-based metrics and alarms

## Common Issues

- symptoms: "No access logs from Envoy"
  diagnosis: "Access logging not configured on the virtual node."
  resolution: "Add logging configuration to the virtual node spec with file path /dev/stdout."

- symptoms: "Access logs present but missing fields"
  diagnosis: "Default log format doesn't include all desired fields."
  resolution: "Customize the access log format to include required fields."

- symptoms: "Access logs not appearing in CloudWatch"
  diagnosis: "Container log driver not configured to send Envoy logs to CloudWatch."
  resolution: "Configure awslogs log driver on the Envoy container."

## Output Format

```yaml
root_cause: "access_logging_issue — <specific_cause>"
evidence:
  - type: logging_config
    content: "<virtual node logging configuration>"
  - type: envoy_logs
    content: "<sample access log entries>"
severity: LOW
mitigation:
  immediate: "Configure access logging on virtual nodes"
  long_term: "Implement centralized log collection and analysis"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Configure access logging on virtual node | GREEN | Observability improvement |
| Ensure container log driver captures output | GREEN | Log configuration |
| Verify log format | GREEN | Configuration check |
| Use structured JSON log format | GREEN | Format improvement |
| Configure CloudWatch Logs agent | GREEN | Log collection setup |

## Escalation Conditions

- Fix requires modifying Envoy proxy config in production
- Fix requires changing routing rules

## Data Sensitivity

- **Access logs**: Request-level data including paths, IPs, status codes — INTERNAL
- **Log configuration**: Virtual node logging settings — INTERNAL
- **CloudWatch log groups**: Centralized log storage — INTERNAL

## Prohibited Actions

- NEVER suggest removing mTLS to fix connectivity
- NEVER log sensitive request headers (Authorization, cookies) without data classification review
- NEVER disable access logging in production without approval

## Phase 3 — Rollback

1. Revert virtual node logging configuration to previous state
2. Remove custom log format if it caused issues
3. Verify Envoy access logs are flowing correctly
4. Restore previous CloudWatch Logs configuration
5. Confirm log retention policies are still in effect

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
