---
title: "A1 — Campaign Creation Failures"
description: "Diagnose campaign creation and configuration issues in Pinpoint"
status: active
severity: HIGH
triggers:
  - "campaign creation failed"
  - "cannot create campaign"
  - "campaign configuration error"
owner: devops-agent
objective: "Identify and resolve campaign creation failures"
context: "Campaign creation requires an active application, configured channel, valid segment, and proper IAM permissions. Common failures: missing channel, empty segment, invalid schedule, or quota exceeded."
---

## Phase 1 — Triage

MUST:
- Check application exists: `aws pinpoint get-app --application-id <app-id>`
- Verify channel is enabled: `aws pinpoint get-channels --application-id <app-id>`
- Check segment exists and has endpoints: `aws pinpoint get-segment --application-id <app-id> --segment-id <seg-id>`
- Review CloudTrail for CreateCampaign errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateCampaign --max-results 10`

SHOULD:
- Verify campaign quota not exceeded: `aws pinpoint get-application-settings --application-id <app-id>`
- Check message template exists if referenced
- Verify schedule configuration is valid

MAY:
- Check for concurrent campaign limits
- Review IAM permissions for mobiletargeting:CreateCampaign

## Phase 2 — Remediate

MUST:
- Enable required channel before campaign creation
- Ensure segment has at least one active endpoint
- Fix schedule to use valid UTC or timezone-aware format

SHOULD:
- Test with a small segment first
- Validate message template rendering

MAY:
- Set up campaign creation automation with pre-validation

## Common Issues

- symptoms: "Campaign created but no messages sent"
  diagnosis: "Channel not enabled or segment has zero active endpoints."
  resolution: "Enable channel. Verify segment has endpoints with valid addresses for the channel."

- symptoms: "Campaign creation returns quota error"
  diagnosis: "Maximum campaigns per application exceeded."
  resolution: "Delete unused campaigns or request quota increase."

## Output Format

```yaml
root_cause: "campaign_creation — <specific_cause>"
evidence:
  - type: app_status
    content: "<application status>"
  - type: channel_config
    content: "<channel configuration>"
severity: HIGH
mitigation:
  immediate: "Fix channel or segment configuration"
  long_term: "Implement campaign creation validation pipeline"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — campaigns target customer segments
- Service Disruption Risk: LOW — creation failures do not affect existing campaigns
- Compliance Impact: HIGH — campaign creation must comply with messaging regulations

## Escalation Conditions
- Campaign creation fails with validation errors
- Campaign targets incorrect customer segments
- Campaign scheduling conflicts with blackout periods
- Campaign content violates messaging regulations
- Multiple campaigns fail to create simultaneously

## Data Sensitivity
- Customer segment data — RESTRICTED
- Campaign content and templates — INTERNAL
- Scheduling configurations — INTERNAL
- Targeting criteria — CONFIDENTIAL
- Never expose customer PII in campaign configurations

## Prohibited Actions
- Do NOT create campaigns targeting customers without consent
- Do NOT bypass campaign content review processes
- Do NOT create campaigns during regulatory blackout periods
- Do NOT include PII in campaign metadata
- Do NOT create campaigns without proper opt-out mechanisms
- Do NOT ignore campaign validation errors

## Phase 3 — Rollback
- Delete incorrectly created campaigns before they execute
- Restore previous campaign configurations
- Revert segment targeting changes
- Verify campaign scheduling after changes
- Document campaign creation issues and compliance checks
- Update campaign creation procedures

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
