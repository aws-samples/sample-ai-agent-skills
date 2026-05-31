---
title: "D2 — Software Updates"
description: "Diagnose software update failures on Outposts Servers"
status: active
severity: HIGH
triggers: ["outpost update failed", "software update stuck", "firmware update", "update not applying"]
owner: devops-agent
objective: "Resolve software and firmware update issues on Outposts Servers"
context: "AWS manages all software updates for Outposts Servers. Updates are delivered via the service link and cannot be applied manually. Update windows can be configured."
---

## Phase 1 — Triage

MUST:
- Check Outpost status: `aws outposts get-outpost --outpost-id <outpost-id> --query '{Status:LifeCycleStatus,Name:Name}'`
- Verify service link connectivity: `aws cloudwatch get-metric-statistics --namespace AWS/Outposts --metric-name ConnectedStatus --dimensions Name=OutpostId,Value=<outpost-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Review CloudTrail for update events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=outposts.amazonaws.com --max-results 20`

SHOULD:
- Check if maintenance window is configured correctly
- Verify no active workloads would be disrupted by the update
- Check AWS Health Dashboard for known update issues

MAY:
- Review update history via AWS Support cases
- Check for pending updates in the Outposts console

## Phase 2 — Remediate

MUST:
- Ensure service link is active and stable before update attempts
- Verify sufficient bandwidth on the service link for update delivery
- Confirm maintenance window allows enough time for update completion

SHOULD:
- Schedule updates during low-traffic periods
- Notify stakeholders before maintenance windows

MAY:
- Configure CloudWatch alarms for service link connectivity during updates
- Plan workload migration during update windows

## Common Issues

- symptoms: "Software update not applying despite active service link"
  diagnosis: "Maintenance window not configured or update requires specific conditions."
  resolution: "Check maintenance window configuration in the Outposts console. Ensure service link bandwidth is sufficient. Contact AWS Support if update is stuck."

## Safety Ratings

```
safety_ratings:
  - "get-outpost: GREEN — read-only status check"
  - "cloudwatch get-metric-statistics: GREEN — read-only metric retrieval"
  - "Maintenance window configuration: YELLOW — affects update scheduling"
  - "Update application: YELLOW — AWS-managed, may cause brief instance impact"
```

## Escalation Conditions

- Update has been pending for more than 7 days with active service link
- Update causes instance instability or unexpected reboots
- Service link drops during an active update
- AWS Health notification indicates a critical security update is required

## Data Sensitivity

- `aws outposts get-outpost`: LOW — returns Outpost metadata
- Update status information: LOW — reveals software version details
- Maintenance window configuration: LOW — reveals scheduling preferences
- Service link bandwidth data: MEDIUM — reveals network capacity

## Prohibited Actions

- NEVER attempt to manually apply firmware or software updates to the Outposts Server
- NEVER SSH into or attempt console access to the server host for update purposes
- NEVER disconnect the service link during an active update
- NEVER skip critical security updates without documented risk acceptance

## Phase 3 — Rollback

1. If an update causes issues, contact AWS Support immediately — rollback is AWS-managed
2. If maintenance window was changed, revert to the previous schedule
3. If workloads were migrated for the update, restore them to the Outpost after update completion
4. Document the update timeline and any issues encountered
5. Verify all instances and volumes are healthy after update completion

## Output Format

```yaml
root_cause: "software_update — <specific_cause>"
evidence:
  - type: update_status
    content: "<update state and service link connectivity>"
severity: HIGH
mitigation:
  immediate: "Resolve update blocker"
  long_term: "Configure maintenance windows and monitoring"
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
