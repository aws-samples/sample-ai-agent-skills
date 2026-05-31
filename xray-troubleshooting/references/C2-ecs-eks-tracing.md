---
title: "C2 — ECS/EKS Tracing Issues"
description: "Diagnose X-Ray tracing problems with ECS and EKS containers"
status: active
severity: HIGH
triggers:
  - "ECS traces missing"
  - "EKS tracing not working"
  - "container tracing"
  - "sidecar daemon"
owner: devops-agent
objective: "Identify and resolve X-Ray tracing issues in containerized environments"
context: "ECS/EKS require the X-Ray daemon as a sidecar container. The daemon must be in the same task (ECS) or pod (EKS). The application container must reach the daemon on UDP 2000. The daemon needs IAM permissions via the task/pod role."
---

## Phase 1 — Triage

MUST:
- Verify X-Ray daemon sidecar is in the task/pod definition
- Check daemon container status and logs
- Verify the task/pod IAM role has X-Ray permissions
- Check if the application can reach the daemon on UDP 2000

SHOULD:
- Verify the daemon container image is correct: `amazon/aws-xray-daemon`
- Check security groups allow UDP 2000 within the task/pod
- Verify the SDK daemon address environment variable: `AWS_XRAY_DAEMON_ADDRESS`

MAY:
- Check ECS task definition or Kubernetes pod spec for daemon configuration
- Verify resource limits on the daemon container

## Phase 2 — Remediate

MUST:
- Add X-Ray daemon as sidecar container in the task/pod definition
- Grant X-Ray permissions to the task/pod IAM role
- Ensure UDP 2000 connectivity between application and daemon containers

SHOULD:
- Set `AWS_XRAY_DAEMON_ADDRESS` environment variable in the application container
- For ECS Fargate, use sidecar in the same task definition
- For EKS, use a DaemonSet or sidecar injection

MAY:
- Configure daemon resource limits (CPU, memory)
- Set up daemon health checks

## Common Issues

- symptoms: "ECS Fargate traces missing"
  diagnosis: "X-Ray daemon sidecar not in the task definition."
  resolution: "Add X-Ray daemon container to the Fargate task definition."

- symptoms: "Daemon running but application cannot connect"
  diagnosis: "Application using wrong daemon address or port."
  resolution: "Set AWS_XRAY_DAEMON_ADDRESS=localhost:2000 in the application container."

- symptoms: "Daemon logs show permission errors"
  diagnosis: "Task role missing X-Ray permissions."
  resolution: "Attach AWSXRayDaemonWriteAccess to the ECS task role."

## Output Format

```yaml
root_cause: "ecs_eks_tracing_issue — <specific_cause>"
evidence:
  - type: daemon_status
    content: "<daemon container status>"
  - type: task_role
    content: "<IAM permissions on task/pod role>"
  - type: connectivity
    content: "<UDP 2000 connectivity status>"
severity: HIGH
mitigation:
  immediate: "Fix daemon sidecar configuration"
  long_term: "Standardize daemon deployment across all container services"
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
