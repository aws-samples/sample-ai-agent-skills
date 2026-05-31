---
title: "Z1 — General EventBridge Advanced Troubleshooting"
description: "Catch-all runbook for EventBridge advanced feature issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "eventbridge problem"
  - "scheduler issue"
  - "pipes issue"
  - "eventbridge error"
  - "eventbridge not working"
owner: devops-agent
objective: "Provide a systematic approach to diagnosing unclassified EventBridge advanced feature issues"
context: "This catch-all runbook covers EventBridge Scheduler, Pipes, Schema Registry, Archive/Replay, and other advanced feature issues that don't clearly fit into specific categories."
---

## Phase 1 — Triage

MUST:
- Identify which EventBridge feature is affected (Scheduler, Pipes, Schema, Archive, Rules)
- Check the feature-specific status and configuration
- Check CloudWatch metrics for the affected feature
- Check CloudTrail for recent API calls and errors

SHOULD:
- Verify IAM permissions for the affected feature
- Check for service health issues in AWS Health Dashboard
- Review recent configuration changes
- Check for related service issues (Lambda, SQS, etc.)

MAY:
- Check service quotas for the affected feature
- Review feature-specific documentation
- Test with simplified configuration to isolate the issue

## Phase 2 — Remediate

MUST:
- Address the identified root cause
- Route to the appropriate specific runbook if identified
- Verify the feature returns to normal operation

SHOULD:
- Document the issue and resolution
- Set up monitoring for similar issues
- Review configuration against best practices

MAY:
- Implement automated health checks
- Create monitoring dashboards for EventBridge features

## Common Issues

- symptoms: "Confusion between Scheduler and EventBridge rules"
  diagnosis: "Scheduler and rules are separate services with different APIs."
  resolution: "Use aws scheduler for Scheduler. Use aws events for rules. They are not interchangeable."

- symptoms: "Events not flowing through the system"
  diagnosis: "Multiple potential causes — event bus, rules, targets, permissions."
  resolution: "Trace the event flow: source → bus → rule → target. Check each stage."

- symptoms: "Feature not available in region"
  diagnosis: "Not all EventBridge features are available in all regions."
  resolution: "Check AWS regional services list for feature availability."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: feature
    content: "<affected feature status and configuration>"
  - type: metrics
    content: "<relevant CloudWatch metrics>"
  - type: cloudtrail
    content: "<recent API events>"
severity: MEDIUM
mitigation:
  immediate: "Address the identified root cause"
  long_term: "Implement comprehensive monitoring for EventBridge features"
```


## Safety Ratings
```
safety_ratings:
  - "Check feature status and configuration: GREEN — read-only API calls"
  - "Check CloudWatch metrics: GREEN — read-only monitoring query"
  - "Check CloudTrail events: GREEN — read-only audit log query"
  - "Verify IAM permissions: GREEN — read-only IAM inspection"
  - "Route to specific runbook: GREEN — diagnostic routing, no state change"
  - "Modify feature configuration: YELLOW — changes depend on the specific feature"
```

## Escalation Conditions
- Issue affects production event processing
- Problem spans multiple EventBridge features (Scheduler, Pipes, Rules)
- Feature not available in the required region
- Service health issue identified in AWS Health Dashboard
- Issue cannot be classified into a specific runbook category

## Data Sensitivity
```
data_sensitivity:
  classification: MEDIUM
  sensitive_fields:
    - "Feature configurations: event processing setup"
    - "CloudTrail events: API call history with parameters"
    - "CloudWatch metrics: operational patterns"
  handling: "CloudTrail events may contain sensitive API parameters. Mask account IDs and ARNs in shared reports."
```

## Prohibited Actions
- NEVER confuse Scheduler API (aws scheduler) with EventBridge Rules API (aws events)
- NEVER make configuration changes without identifying the specific affected feature first
- NEVER assume feature availability across all regions without checking
- NEVER modify multiple features simultaneously when troubleshooting

## Phase 3 — Rollback
- If configuration was changed: revert to previous configuration for the specific feature
- If IAM permissions were modified: restore previous IAM policy version
- If feature was enabled/disabled: restore previous state
- For feature-specific rollback: refer to the appropriate specific runbook (A1-G2)

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
