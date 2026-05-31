---
title: "A3 — Campaign Scheduling Issues"
description: "Diagnose campaign scheduling and timing problems"
status: active
severity: MEDIUM
triggers:
  - "campaign fired at wrong time"
  - "campaign not starting"
  - "schedule not working"
  - "recurring campaign issues"
owner: devops-agent
objective: "Identify and resolve campaign scheduling issues"
context: "Campaign schedules use UTC by default. Timezone mismatches, invalid cron expressions, and quiet time settings cause unexpected behavior."
---

## Phase 1 — Triage

MUST:
- Check campaign schedule config: `aws pinpoint get-campaign --application-id <app-id> --campaign-id <campaign-id> --query 'CampaignResponse.Schedule'`
- Verify campaign state is ACTIVE: `aws pinpoint get-campaign --application-id <app-id> --campaign-id <campaign-id> --query 'CampaignResponse.State'`
- Check application quiet time settings: `aws pinpoint get-application-settings --application-id <app-id>`
- Review campaign activities for execution history: `aws pinpoint get-campaign-activities --application-id <app-id> --campaign-id <campaign-id>`

SHOULD:
- Verify timezone setting matches intended delivery time
- Check if quiet time is suppressing delivery
- Confirm start/end dates are in the future

MAY:
- Check for overlapping campaign schedules
- Review CloudTrail for schedule modification events

## Phase 2 — Remediate

MUST:
- Set explicit timezone in campaign schedule
- Ensure start time is in the future for new campaigns
- Adjust quiet time settings if blocking delivery

SHOULD:
- Use local time delivery for multi-timezone audiences
- Test schedule with a small segment first

MAY:
- Implement schedule validation in deployment pipeline

## Common Issues

- symptoms: "Campaign fires 5 hours early/late"
  diagnosis: "Schedule uses UTC but user expected local time."
  resolution: "Set explicit timezone in schedule configuration."

- symptoms: "Recurring campaign stops executing"
  diagnosis: "Campaign end date reached or campaign paused."
  resolution: "Check end date. Verify campaign state is ACTIVE."

## Output Format

```yaml
root_cause: "campaign_scheduling — <specific_cause>"
evidence:
  - type: schedule_config
    content: "<schedule configuration>"
  - type: campaign_state
    content: "<campaign state>"
severity: MEDIUM
mitigation:
  immediate: "Fix timezone or schedule configuration"
  long_term: "Standardize campaign scheduling with explicit timezones"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — scheduling is a configuration concern
- Service Disruption Risk: MEDIUM — scheduling issues can cause missed or duplicate sends
- Compliance Impact: HIGH — scheduling must respect quiet hours and blackout periods

## Escalation Conditions
- Campaigns execute at wrong times
- Scheduling causes duplicate message sends
- Quiet hours not being respected
- Time zone handling causes incorrect delivery times
- Scheduled campaigns fail to execute

## Data Sensitivity
- Schedule configurations — INTERNAL
- Time zone settings — INTERNAL
- Quiet hours configurations — INTERNAL
- Campaign execution logs — INTERNAL
- Never expose scheduling details that reveal campaign strategy

## Prohibited Actions
- Do NOT schedule campaigns during regulatory blackout periods
- Do NOT ignore quiet hours configurations
- Do NOT schedule overlapping campaigns to same segments
- Do NOT bypass scheduling validation
- Do NOT modify active campaign schedules without approval
- Do NOT ignore scheduling failures

## Phase 3 — Rollback
- Cancel incorrectly scheduled campaigns
- Restore previous scheduling configurations
- Revert time zone changes
- Verify scheduling accuracy after changes
- Document scheduling issues and compliance impact
- Update scheduling procedures

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
