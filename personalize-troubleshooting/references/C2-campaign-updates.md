---
title: "C2 — Campaign Update Issues"
description: "Diagnose campaign update and scaling problems"
status: active
severity: MEDIUM
triggers: ["campaign update failed", "cannot update campaign", "TPS scaling error"]
owner: devops-agent
objective: "Identify and resolve campaign update issues"
context: "Campaigns can be updated to point to new solution versions or change MinProvisionedTPS. Updates are not instantaneous."
---

## Phase 1 — Triage

MUST:
- Check campaign: `aws personalize describe-campaign --campaign-arn <arn>`
- Verify campaign is in ACTIVE status (not updating)
- Check latestCampaignUpdate for update status
- Verify new solution version is ACTIVE

SHOULD:
- Check if previous update is still in progress
- Verify MinProvisionedTPS change is valid
- Monitor campaign during update

MAY:
- Check CloudTrail for UpdateCampaign events
- Review campaign utilization metrics

## Phase 2 — Remediate

MUST:
- Wait for current update to complete before new update
- Ensure new solution version is ACTIVE
- Verify campaign returns to ACTIVE after update

SHOULD:
- Test new solution version before updating production campaign
- Monitor recommendations after update

MAY:
- Implement blue/green campaign deployment

## Common Issues

- symptoms: "UpdateCampaign fails"
  diagnosis: "Campaign already updating or new solution version not ACTIVE."
  resolution: "Wait for current update. Verify solution version status."

- symptoms: "Recommendations change after campaign update"
  diagnosis: "New solution version produces different recommendations."
  resolution: "Expected behavior. Test new version before production update."

## Output Format

```yaml
root_cause: "campaign_update — <specific_cause>"
evidence:
  - type: campaign_status
    content: "<campaign and update details>"
severity: MEDIUM
mitigation:
  immediate: "Wait for update or fix configuration"
  long_term: "Implement campaign update pipeline with testing"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — campaign updates modify serving configuration
- Service Disruption Risk: MEDIUM — failed updates can affect recommendation availability
- Compliance Impact: LOW — campaign updates are operational

## Escalation Conditions
- Campaign update fails leaving campaign in degraded state
- Update causes recommendation quality degradation
- Update causes latency increase
- Throughput changes cause throttling
- Multiple campaign updates fail simultaneously

## Data Sensitivity
- Campaign update parameters — INTERNAL
- Solution version details — INTERNAL
- Throughput configurations — INTERNAL
- Update logs — INTERNAL
- Never expose campaign internal configurations

## Prohibited Actions
- Do NOT update campaigns during peak traffic without approval
- Do NOT reduce throughput below minimum required for traffic
- Do NOT update to untested solution versions
- Do NOT ignore campaign update failures
- Do NOT bypass campaign update validation
- Do NOT update multiple campaigns simultaneously without coordination

## Phase 3 — Rollback
- Revert campaign to previous solution version
- Restore previous throughput settings
- Verify recommendation quality after rollback
- Restore monitoring configurations
- Document update issues and resolution steps
- Update campaign management procedures

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
