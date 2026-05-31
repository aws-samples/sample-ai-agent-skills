---
title: "F2 — Action Item Issues"
description: "Diagnose Incident Manager action item tracking problems including creation and follow-up"
status: active
severity: LOW
triggers:
  - "action items"
  - "follow-up actions"
  - "action item tracking"
  - "incident action"
  - "remediation tracking"
owner: devops-agent
objective: "Resolve action item tracking issues and ensure follow-up completion"
context: "Action items are created during post-incident analysis to track remediation tasks. They can be linked to Jira, ServiceNow, or other systems via related items. Common issues include action items not being created, tracking integration failures, and incomplete follow-up."
---

## Phase 1 — Triage

MUST:
- Check incident related items: `aws ssm-incidents list-related-items --incident-record-arn <incident-arn>`
- Verify incident has been through post-incident analysis
- Check if action items were created during analysis
- Verify permissions for creating and updating related items

SHOULD:
- Check if integration with external tracking systems is configured
- Verify action item assignees and due dates
- Review past incidents for action item patterns

MAY:
- Add related items manually: `aws ssm-incidents update-related-items --incident-record-arn <incident-arn> --related-items-update '{"itemToAdd":{"identifier":{"type":"OTHER","value":{"url":"<url>"}},"title":"Action Item"}}'`
- Check CloudTrail for related item events

## Phase 2 — Remediate

MUST:
- Create missing action items as related items
- Link to external tracking systems (Jira, ServiceNow) via URLs
- Assign owners and due dates

SHOULD:
- Establish a standard action item template
- Schedule follow-up reviews for open action items

## Output Format

```yaml
root_cause: "action_items — <specific_cause>"
evidence:
  - type: related_items
    content: "<incident related items>"
  - type: analysis_status
    content: "<post-incident analysis status>"
severity: LOW
mitigation:
  immediate: "Create and track action items"
  long_term: "Integrate with external tracking systems and establish review cadence"
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
- Never delete response plans that are actively referenced by CloudWatch alarms
- Never modify escalation plans during active incidents
- Never remove contact channels without confirming alternative notification paths exist
- Never delete incident records or post-incident analysis data
- Never disable automation runbooks tied to active response plans without backup procedures

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous response plan configuration if incident detection breaks
- Revert escalation plan changes if notifications fail to reach responders
- Re-enable previous contact channel settings if engagement plans stop working

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
