---
title: "C1 — Organizational Health"
description: "Diagnose organizational health view issues"
status: active
severity: MEDIUM
triggers:
  - "organizational health not working"
  - "org health view"
  - "multi-account health"
  - "cross-account health events"
owner: devops-agent
objective: "Resolve organizational health view configuration issues"
context: "Organizational health aggregates health events across all accounts in an AWS Organization. Requires Organizations with all features enabled and Health enabled as a trusted service. Accessible from management account or delegated admin."
---

## Phase 1 — Triage

MUST:
- Verify Organizations has all features enabled: `aws organizations describe-organization --query 'Organization.FeatureSet'`
- Check if Health is enabled as trusted service: `aws organizations list-aws-service-access-for-organization --query 'EnabledServicePrincipals[?ServicePrincipal==\`health.amazonaws.com\`]'`
- Verify calling from management account or delegated admin
- Test organizational health API: `aws health describe-events-for-organization --filter '{"eventStatusCodes":["open"]}' --region us-east-1`

SHOULD:
- Check affected accounts for an event: `aws health describe-affected-accounts-for-organization --event-arn <event-arn> --region us-east-1`
- Verify IAM permissions include health:Describe*ForOrganization
- Check if specific accounts are excluded

MAY:
- Review organizational health event history
- Check delegated administrator configuration

## Phase 2 — Remediate

MUST:
- Enable all features in Organizations (if only consolidated billing)
- Enable Health as trusted service: `aws organizations enable-aws-service-access --service-principal health.amazonaws.com`
- Access from management account or register delegated admin

SHOULD:
- Set up EventBridge rules for organizational health events
- Create dashboards for cross-account health visibility
- Grant appropriate IAM permissions for org health access

MAY:
- Register delegated admin for Health
- Implement automated cross-account incident response

## Common Issues

- symptoms: "Organizational health API returns empty results"
  diagnosis: "Health not enabled as trusted service or not calling from management account."
  resolution: "Enable Health service access. Use management account or delegated admin."

- symptoms: "Cannot see member account events"
  diagnosis: "Organizations using consolidated billing only (not all features)."
  resolution: "Enable all features in Organizations."

- symptoms: "SubscriptionRequiredException on org health API"
  diagnosis: "Management account needs Business or Enterprise Support for Health API."
  resolution: "Upgrade management account support plan."

## Output Format

```yaml
root_cause: "organizational_health — <specific_cause>"
evidence:
  - type: org_config
    content: "<Organizations feature set and trusted services>"
  - type: health_access
    content: "<management account or delegated admin status>"
severity: MEDIUM
mitigation:
  immediate: "Enable all features and Health trusted service"
  long_term: "Set up organizational health monitoring and dashboards"
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
