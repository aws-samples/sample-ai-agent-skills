---
title: "Z1 — General Incident Manager Troubleshooting (Catch-All)"
description: "Fallback SOP for Incident Manager issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown Incident Manager issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Check replication set: `aws ssm-incidents get-replication-set`
- List response plans: `aws ssm-incidents list-response-plans`
- List incidents: `aws ssm-incidents list-incident-records`
- List contacts: `aws ssm-contacts list-contacts`
- Check recent events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ssm-incidents.amazonaws.com --max-results 10`

SHOULD:
- Verify IAM permissions
- Check active engagements: `aws ssm-contacts list-engagements`
- Verify CloudWatch alarm integrations

MAY:
- Check AWS Health Dashboard for Incident Manager service events
- Review EventBridge rules targeting Incident Manager

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Response plan issues → Response (A1)
- Incident creation failures → Response (A2)
- Escalation problems → Response (A3)
- Contact channel issues → Contacts (B1)
- Engagement plan problems → Contacts (B2)
- Automation runbook failures → Runbooks (C1)
- Manual step issues → Runbooks (C2)
- Timeline event problems → Timeline (D1)
- Metrics issues → Timeline (D2)
- CloudWatch alarm integration → Integration (E1)
- EventBridge integration → Integration (E2)
- Post-incident analysis → Post-Incident (F1)
- Action item tracking → Post-Incident (F2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all response plan configurations
- Review CloudTrail for recent changes
- Verify all contacts and channels
- Test basic operations

SHOULD:
- Compare with known-good configuration
- Check AWS Health Dashboard

MAY:
- Check for regional service outages
- Review Incident Manager console for visual indicators

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<response|contacts|runbooks|timeline|integration|post_incident|unknown>"
investigation_path: "replication set → response plans → contacts → <domain_classification>"
evidence:
  - type: replication_set
    content: "<replication set status>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
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
