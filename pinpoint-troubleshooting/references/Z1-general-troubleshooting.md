---
title: "Z1 — General Pinpoint Troubleshooting"
description: "Catch-all runbook for Pinpoint issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "pinpoint error"
  - "pinpoint not working"
  - "pinpoint issue"
  - "something wrong with pinpoint"
owner: devops-agent
objective: "Systematically diagnose unspecified Pinpoint issues"
context: "Use this runbook when the issue doesn't clearly map to a specific runbook. Follows a systematic approach to identify the problem domain."
---

## Phase 1 — Triage

MUST:
- List applications: `aws pinpoint get-apps`
- Check application settings: `aws pinpoint get-application-settings --application-id <app-id>`
- Check all channels: `aws pinpoint get-channels --application-id <app-id>`
- Review recent CloudTrail events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=pinpoint.amazonaws.com --max-results 20`

SHOULD:
- Check for recent campaigns: `aws pinpoint get-campaigns --application-id <app-id>`
- Check for active journeys: `aws pinpoint get-journeys --application-id <app-id>`
- Review CloudWatch metrics for anomalies
- Check IAM permissions for the calling identity

MAY:
- Check service health dashboard for Pinpoint
- Review account-level quotas
- Check event stream configuration

## Phase 2 — Remediate

MUST:
- Route to specific runbook once problem domain identified
- Document findings for future reference

SHOULD:
- Check multiple problem domains if initial investigation inconclusive
- Verify IAM permissions are sufficient

MAY:
- Engage AWS Support if issue persists after systematic investigation

## Common Issues

- symptoms: "General Pinpoint errors without clear cause"
  diagnosis: "Start with application health, then channels, then campaigns/journeys."
  resolution: "Follow systematic triage. Route to specific runbook."

- symptoms: "API calls returning AccessDeniedException"
  diagnosis: "IAM permissions insufficient for Pinpoint operations."
  resolution: "Add mobiletargeting:* permissions or specific required actions."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: application_status
    content: "<application health>"
  - type: channel_status
    content: "<channel configurations>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook"
  long_term: "Implement comprehensive Pinpoint monitoring"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — troubleshooting may involve customer data configurations
- Service Disruption Risk: LOW — diagnostic operations are read-only
- Compliance Impact: LOW — standard troubleshooting procedures

## Escalation Conditions
- Issue cannot be resolved using standard troubleshooting procedures
- Pinpoint service becomes unavailable
- Multiple unrelated issues occur simultaneously
- AWS Health Dashboard shows Pinpoint service event
- Issue requires AWS Support engagement

## Data Sensitivity
- Service configurations — INTERNAL
- Customer data references — RESTRICTED
- Delivery metrics — INTERNAL
- Error messages — INTERNAL
- Never expose customer PII during troubleshooting

## Prohibited Actions
- Do NOT inspect individual customer data without privacy safeguards
- Do NOT make changes to production configurations without approval
- Do NOT share service details in public forums without sanitizing
- Do NOT bypass established escalation procedures
- Do NOT attempt fixes outside Pinpoint service boundaries
- Do NOT ignore recurring issues

## Phase 3 — Rollback
- Revert any diagnostic changes made during troubleshooting
- Restore service to last known good configuration
- Document all troubleshooting steps and outcomes
- Verify service health after troubleshooting
- Update runbooks with new findings
- Close out temporary access granted for troubleshooting

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
