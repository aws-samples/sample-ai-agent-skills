---
title: "G2 — SaaS Event Delivery Issues"
description: "Diagnose SaaS event delivery and processing problems"
status: active
severity: MEDIUM
triggers:
  - "SaaS events"
  - "SaaS not delivering"
  - "third-party events"
  - "SaaS integration"
  - "partner events missing"
owner: devops-agent
objective: "Identify and resolve SaaS event delivery and processing issues"
context: "SaaS events arrive via partner event sources to dedicated event buses. Issues can occur at the SaaS provider, during delivery to EventBridge, or in rule processing. Each SaaS provider has specific event formats and delivery mechanisms."
---

## Phase 1 — Triage

MUST:
- Verify partner event bus is receiving events: `aws events list-event-buses --name-prefix aws.partner`
- Check rules on the partner event bus: `aws events list-rules --event-bus-name <partner-bus-name>`
- Verify rule targets are configured and healthy
- Check CloudWatch metrics for the partner event bus

SHOULD:
- Verify SaaS provider is sending events (check provider dashboard)
- Check event pattern in rules matches SaaS event format
- Review target invocation metrics
- Check DLQ for failed event deliveries

MAY:
- Review SaaS provider documentation for event format
- Test with SaaS provider's event simulation tools
- Check for SaaS provider outages

## Phase 2 — Remediate

MUST:
- Verify SaaS provider configuration and event delivery
- Fix rule event patterns to match SaaS event format
- Ensure targets are healthy and accessible

SHOULD:
- Configure DLQ on rules for failed event capture
- Set up monitoring for SaaS event delivery
- Document SaaS event format and processing logic

MAY:
- Implement SaaS event validation
- Set up alerts for SaaS event delivery gaps

## Common Issues

- symptoms: "SaaS events arriving but rules not matching"
  diagnosis: "Rule event pattern does not match the SaaS event structure."
  resolution: "Review actual SaaS event format. Update rule event pattern to match."

- symptoms: "SaaS events stopped arriving"
  diagnosis: "SaaS provider configuration changed or subscription expired."
  resolution: "Check SaaS provider dashboard. Verify API keys and webhook configuration."

- symptoms: "Duplicate SaaS events being processed"
  diagnosis: "SaaS provider sends events with at-least-once delivery guarantee."
  resolution: "Implement idempotent event processing using event IDs."

## Output Format

```yaml
root_cause: "saas_events — <specific_cause>"
evidence:
  - type: partner_bus
    content: "<partner event bus and rules>"
  - type: saas_provider
    content: "<SaaS provider configuration status>"
  - type: events
    content: "<event delivery and processing metrics>"
severity: MEDIUM
mitigation:
  immediate: "Fix SaaS configuration or rule patterns"
  long_term: "Implement SaaS event monitoring and idempotent processing"
```


## Safety Ratings
```
safety_ratings:
  - "Verify partner event bus: GREEN — read-only API call"
  - "Check rules on partner bus: GREEN — read-only API call"
  - "Check CloudWatch metrics: GREEN — read-only monitoring query"
  - "Update rule event patterns: YELLOW — changes which SaaS events are processed"
  - "Modify target configuration: YELLOW — changes event delivery destination"
  - "Configure DLQ on rules: GREEN — adds error handling without changing processing"
```

## Escalation Conditions
- SaaS integration serves production business processes
- SaaS events stopped arriving requiring vendor coordination
- Rule patterns not matching changed SaaS event format
- Duplicate SaaS events causing data integrity issues
- SaaS provider outage affecting event delivery

## Data Sensitivity
```
data_sensitivity:
  classification: MEDIUM
  sensitive_fields:
    - "SaaS event payloads: third-party business data"
    - "SaaS provider configuration: API keys, webhook URLs"
    - "Rule event patterns: business logic for SaaS event processing"
  handling: "SaaS event payloads may contain third-party business data. Do not expose provider credentials."
```

## Prohibited Actions
- NEVER share SaaS provider API keys or webhook URLs in logs or outputs
- NEVER delete rules processing SaaS events without confirming downstream impact
- NEVER modify SaaS provider configuration without vendor coordination
- NEVER assume SaaS event format is stable — always verify against actual events

## Phase 3 — Rollback
- If rule event pattern was changed: restore previous event pattern
- If target was modified: revert to previous target configuration
- If DLQ was configured: remove DLQ configuration if not needed
- If SaaS provider configuration was changed: coordinate with vendor to restore previous settings
- If partner event bus rules were deleted: recreate rules from IaC or documentation

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
