---
title: "D1 — Timeline Event Issues"
description: "Diagnose Incident Manager timeline event problems including missing events and incorrect timestamps"
status: active
severity: MEDIUM
triggers:
  - "timeline events"
  - "missing timeline"
  - "incident timeline"
  - "event not showing"
  - "timeline error"
owner: devops-agent
objective: "Resolve timeline event issues and ensure complete incident documentation"
context: "Incident Manager automatically records timeline events for incident lifecycle changes, engagements, and automation steps. Custom events can be added manually. Common issues include missing events, replication delays, and custom event creation failures."
---

## Phase 1 — Triage

MUST:
- List timeline events: `aws ssm-incidents list-timeline-events --incident-record-arn <incident-arn>`
- Check incident record status: `aws ssm-incidents get-incident-record --arn <incident-arn>`
- Verify the incident is in the correct region
- Check if events are delayed due to cross-region replication

SHOULD:
- Verify automation execution events are being recorded
- Check if custom events were created with correct timestamps
- Review related items: `aws ssm-incidents list-related-items --incident-record-arn <incident-arn>`

MAY:
- Add a custom timeline event: `aws ssm-incidents create-timeline-event --incident-record-arn <incident-arn> --event-time <timestamp> --event-type "Custom Event"`
- Check CloudTrail for timeline event creation

## Phase 2 — Remediate

MUST:
- If events missing due to replication delay, wait and check again
- If custom events needed, create them manually
- Verify the incident record is accessible

SHOULD:
- Add custom events for important actions not automatically tracked
- Document timeline event expectations for the team

## Output Format

```yaml
root_cause: "timeline_events — <specific_cause>"
evidence:
  - type: timeline
    content: "<timeline events list>"
  - type: incident_status
    content: "<incident record status>"
severity: MEDIUM
mitigation:
  immediate: "Wait for replication or add custom events"
  long_term: "Establish timeline documentation standards"
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
