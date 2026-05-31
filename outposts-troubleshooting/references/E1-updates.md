---
title: "E1 — Outpost Update Failures"
description: "Diagnose firmware and software update issues on AWS Outposts"
status: active
severity: MEDIUM
triggers:
  - "outpost update"
  - "firmware update"
  - "software update"
  - "maintenance window"
owner: devops-agent
objective: "Resolve Outpost update failures and ensure successful maintenance operations"
context: "AWS manages Outpost firmware and software updates. Updates require service link connectivity and may require brief instance reboots. AWS provides advance notification for updates that impact instances."
---

## Phase 1 — Triage

MUST:
- Check Outpost status: `aws outposts get-outpost --outpost-id <outpost-id>`
- Verify service link connectivity: `aws cloudwatch get-metric-statistics --namespace AWS/Outposts --metric-name ConnectedStatus --dimensions Name=OutpostId,Value=<outpost-id> --start-time <start> --end-time <end> --period 60 --statistics Average`
- Check for scheduled events: `aws ec2 describe-instance-status --filters "Name=event.code,Values=system-maintenance,instance-retirement" --query 'InstanceStatuses[?OutpostArn!=null]'`
- Review AWS Health events: `aws health describe-events --filter "services=OUTPOSTS,eventTypeCategories=scheduledChange"`

SHOULD:
- Check CloudTrail for update-related events
- Verify no ongoing capacity issues that could block updates
- Review notification history for update schedules

MAY:
- Contact AWS Support for update status details
- Review update history and patterns

## Phase 2 — Remediate

MUST:
- Ensure service link is connected — updates cannot proceed without it
- If instances need rebooting: plan maintenance window and notify stakeholders
- Do not block AWS management traffic required for updates

SHOULD:
- Prepare instance migration plans for maintenance events
- Create snapshots before scheduled maintenance
- Monitor update progress through AWS Health Dashboard

MAY:
- Request update scheduling adjustments through AWS Support
- Implement automated instance migration for maintenance events

## Common Issues

- symptoms: "Update stuck or not progressing"
  diagnosis: "Service link connectivity issues preventing update delivery."
  resolution: "Verify service link is connected. Check firewall rules and network path."

- symptoms: "Instances rebooted unexpectedly"
  diagnosis: "Scheduled maintenance event for firmware/software update."
  resolution: "Check AWS Health events. Plan for future maintenance windows."

## Output Format

```yaml
root_cause: "update — <specific_cause>"
evidence:
  - type: outpost_status
    content: "<Outpost status>"
  - type: health_events
    content: "<scheduled maintenance events>"
severity: MEDIUM
mitigation:
  immediate: "Resolve update blocker or plan maintenance window"
  long_term: "Implement maintenance notification and migration automation"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: LOW — updates are operational maintenance
- Service Disruption Risk: HIGH — failed updates can affect Outpost availability
- Compliance Impact: MEDIUM — updates may include security patches

## Escalation Conditions
- Update fails leaving Outpost in degraded state
- Update causes unexpected workload disruption
- Update rollback fails
- Security patch update is critical but cannot be applied
- Update requires extended maintenance window

## Data Sensitivity
- Update schedules and configurations — INTERNAL
- Firmware versions — INTERNAL
- Patch details — INTERNAL
- Maintenance window configurations — INTERNAL
- Never expose update schedules that reveal vulnerability windows

## Prohibited Actions
- Do NOT skip critical security updates
- Do NOT apply updates without a maintenance window
- Do NOT ignore update failure notifications
- Do NOT apply updates during peak workload periods without approval
- Do NOT modify update configurations without AWS guidance
- Do NOT bypass update validation procedures

## Phase 3 — Rollback
- Contact AWS Support if update rollback is needed
- Migrate workloads if update leaves Outpost in degraded state
- Restore previous configurations if update changed settings
- Verify workload health after update completion or rollback
- Document update issues and timeline
- Update maintenance procedures based on findings

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
