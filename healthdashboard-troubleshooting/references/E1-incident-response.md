---
title: "E1 — Incident Response"
description: "Coordinate incident response using Health Dashboard events"
status: active
severity: HIGH
triggers:
  - "service incident"
  - "AWS outage"
  - "incident response"
  - "service disruption"
owner: devops-agent
objective: "Coordinate effective incident response using Health Dashboard information"
context: "Health Dashboard provides AWS-side incident information. Issue-category events indicate active service disruptions. Events include affected services, regions, and timeline. Use Health events alongside your own monitoring for comprehensive incident response."
---

## Phase 1 — Triage

MUST:
- Check for active issues: `aws health describe-events --filter '{"eventTypeCategories":["issue"],"eventStatusCodes":["open"]}' --region us-east-1`
- Get event details for active issues: `aws health describe-event-details --event-arns <event-arn> --region us-east-1`
- Identify affected resources: `aws health describe-affected-entities --filter '{"eventArns":["<event-arn>"]}' --region us-east-1`
- Correlate health events with your monitoring alerts

SHOULD:
- Check AWS status page for broader context
- Verify if your resources are in the affected region/AZ
- Check CloudWatch metrics for your resources in the affected service

MAY:
- Review related events for the same service
- Check if the issue affects multiple services

## Phase 2 — Remediate

MUST:
- Assess impact on your workloads based on affected resources
- Activate incident response procedures if production is impacted
- Communicate status to stakeholders with event details and timeline

SHOULD:
- Implement failover to unaffected regions/AZs if available
- Monitor event updates for resolution timeline
- Document incident timeline and actions taken

MAY:
- Automate failover based on health event triggers
- Create post-incident review using health event timeline

## Common Issues

- symptoms: "Service disruption but no health event"
  diagnosis: "Health events have publication delay. Issue may not be confirmed yet."
  resolution: "Use your own monitoring. Check AWS status page. Health event may appear later."

- symptoms: "Health event says resolved but still experiencing issues"
  diagnosis: "AWS-side issue resolved but your resources may need recovery actions."
  resolution: "Restart affected resources. Clear caches. Verify connectivity."

- symptoms: "Multiple health events during incident"
  diagnosis: "Complex incidents may generate multiple events for different aspects."
  resolution: "Track all related events. Each may have different affected resources and timelines."

## Output Format

```yaml
root_cause: "incident_response — <specific_cause>"
evidence:
  - type: health_events
    content: "<active issue events and details>"
  - type: impact_assessment
    content: "<affected resources and workload impact>"
severity: HIGH
mitigation:
  immediate: "Activate failover and communicate to stakeholders"
  long_term: "Implement automated incident response and multi-region resilience"
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
- Never dismiss or archive health events without documenting their resolution
- Never disable EventBridge rules for health notifications in production accounts
- Never ignore scheduled maintenance events without planning for impact
- Never delete organizational health view configurations without stakeholder approval
- Never modify health event filters that may suppress critical notifications

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Re-enable EventBridge notification rules if health alerts stop flowing
- Restore previous event filter configurations if critical events are missed
- Revert organizational view delegation changes if visibility is lost

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
