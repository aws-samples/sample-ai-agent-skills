---
title: "F1 — Post-Incident Analysis Issues"
description: "Diagnose Incident Manager post-incident analysis problems including availability and data completeness"
status: active
severity: MEDIUM
triggers:
  - "post-incident analysis"
  - "incident analysis"
  - "analysis not available"
  - "incident review"
  - "lessons learned"
owner: devops-agent
objective: "Resolve post-incident analysis issues and ensure complete incident review"
context: "Post-incident analysis is available after an incident is resolved. It includes timeline review, metrics analysis, questions, and action items. Common issues include analysis not available (incident not resolved), incomplete data, and permission errors."
---

## Phase 1 — Triage

MUST:
- Check incident status: `aws ssm-incidents get-incident-record --arn <incident-arn>`
- Verify incident is RESOLVED (analysis requires resolved status)
- Check if 24 hours have passed since resolution (data finalization)
- Verify IAM permissions for analysis operations

SHOULD:
- Check timeline completeness: `aws ssm-incidents list-timeline-events --incident-record-arn <incident-arn>`
- Verify metrics data is available for the incident period
- Check related items are properly linked

MAY:
- Check CloudTrail for analysis-related events
- Verify the response plan had analysis template configured

## Phase 2 — Remediate

MUST:
- If incident not resolved, resolve it first: `aws ssm-incidents update-incident-record --arn <incident-arn> --status RESOLVED`
- Wait up to 24 hours for analysis data to finalize
- Fix IAM permissions if access denied

SHOULD:
- Add custom timeline events before starting analysis
- Link all related items to the incident record

## Output Format

```yaml
root_cause: "post_incident_analysis — <specific_cause>"
evidence:
  - type: incident_status
    content: "<incident record status>"
  - type: timeline_completeness
    content: "<timeline event count>"
severity: MEDIUM
mitigation:
  immediate: "Resolve incident and wait for data finalization"
  long_term: "Establish post-incident analysis process and templates"
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
