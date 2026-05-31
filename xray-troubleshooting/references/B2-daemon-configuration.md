---
title: "B2 — Daemon Configuration Issues"
description: "Diagnose X-Ray daemon configuration and connectivity problems"
status: active
severity: HIGH
triggers:
  - "daemon not running"
  - "daemon not forwarding"
  - "UDP 2000 unreachable"
  - "daemon configuration"
owner: devops-agent
objective: "Identify and resolve X-Ray daemon configuration issues"
context: "The X-Ray daemon receives segments from the SDK on UDP port 2000 and forwards them to the X-Ray API. It must be running and reachable for non-Lambda services. The daemon needs IAM permissions to call X-Ray APIs."
---

## Phase 1 — Triage

MUST:
- Verify the daemon is running (check process or container status)
- Check daemon logs for errors: `aws logs filter-log-events --log-group-name <daemon-log-group> --filter-pattern "ERROR"`
- Verify the daemon can reach the X-Ray API (network/IAM)
- Check if the application can reach the daemon on UDP 2000

SHOULD:
- Verify the daemon's IAM role has xray:PutTraceSegments and xray:PutTelemetryRecords permissions
- Check security groups allow UDP 2000 between application and daemon
- Verify the daemon address configured in the SDK matches the actual daemon location

MAY:
- Check daemon resource usage (CPU, memory)
- Verify the daemon version is current

## Phase 2 — Remediate

MUST:
- Start the daemon if not running
- Fix IAM permissions on the daemon's role: attach AWSXRayDaemonWriteAccess policy
- Ensure network connectivity on UDP 2000 between application and daemon

SHOULD:
- For ECS, add daemon as sidecar in task definition with proper port mapping
- Configure the SDK daemon address if not using localhost: `AWS_XRAY_DAEMON_ADDRESS=<host>:2000`
- Set up daemon health monitoring

MAY:
- Configure daemon logging for troubleshooting
- Adjust daemon buffer size for high-throughput applications

## Common Issues

- symptoms: "Daemon running but traces not appearing"
  diagnosis: "Daemon lacks IAM permissions to call X-Ray API."
  resolution: "Attach AWSXRayDaemonWriteAccess policy to the daemon's IAM role."

- symptoms: "Application logs show 'Failed to send segment'"
  diagnosis: "Daemon not reachable on UDP 2000."
  resolution: "Check daemon is running, security groups allow UDP 2000, and SDK daemon address is correct."

- symptoms: "ECS task traces missing"
  diagnosis: "Daemon sidecar not in the task definition or wrong port configuration."
  resolution: "Add X-Ray daemon container to the task definition with UDP 2000 port mapping."

## Output Format

```yaml
root_cause: "daemon_configuration_issue — <specific_cause>"
evidence:
  - type: daemon_status
    content: "<daemon running status>"
  - type: daemon_logs
    content: "<daemon error logs>"
  - type: network_connectivity
    content: "<UDP 2000 reachability>"
severity: HIGH
mitigation:
  immediate: "Fix daemon configuration and connectivity"
  long_term: "Automate daemon deployment with IaC"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Trace data reveals application request flow and dependencies
- Segment metadata may contain request/response details
- Service maps expose complete application architecture
- Classification: CONFIDENTIAL - application architecture and request flow data

## Prohibited Actions

- NEVER modify sampling rules that could cause production trace data loss
- NEVER delete X-Ray groups used by production monitoring dashboards
- NEVER change encryption settings without coordinating with security team
- NEVER disable tracing on production services without approval

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
