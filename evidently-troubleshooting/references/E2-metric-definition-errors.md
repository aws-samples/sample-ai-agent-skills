---
title: "E2 — Metric Definition Errors"
description: "Diagnose metric definition and event collection issues"
status: active
severity: HIGH
triggers: ["metric not collecting", "metric definition error", "PutProjectEvents failed", "no metric data"]
owner: devops-agent
objective: "Resolve metric definition and event collection issues for experiments"
context: "Metrics are defined in experiments using event patterns. Metric events are sent via PutProjectEvents API. Entity IDs must match between evaluations and events."
---

## Phase 1 — Triage

MUST:
- Get experiment metric goals: `aws evidently get-experiment --project <name> --experiment <experiment-name>`
- Verify PutProjectEvents is being called: check CloudTrail for PutProjectEvents calls
- Check the event structure matches the metric goal's event pattern

SHOULD:
- Verify entity IDs in metric events match entity IDs in EvaluateFeature calls
- Check the metric value key and entity ID key in the metric goal definition
- Verify the project has data delivery configured to inspect raw events

MAY:
- Test PutProjectEvents with a sample event: `aws evidently put-project-events --project <name> --events '[{"timestamp":"<time>","type":"aws.evidently.custom","data":"{\"details\":{\"<valueKey>\":<value>},\"userDetails\":{\"entityId\":\"<entity-id>\"}}"}]'`
- Check data delivery destination for raw event data

## Phase 2 — Remediate

MUST:
- Fix metric goal definition to match the actual event structure
- Ensure PutProjectEvents is called with correct event format
- Verify entity ID consistency between evaluations and metric events

SHOULD:
- Test metric event collection with a known entity before relying on results
- Document the expected event structure for each metric goal

MAY:
- Set up monitoring for PutProjectEvents error rates
- Configure data delivery for debugging metric event issues

## Common Issues

- symptoms: "PutProjectEvents succeeds but metrics don't appear in experiment results"
  diagnosis: "Entity IDs in metric events may not match entity IDs used in EvaluateFeature, or the event pattern doesn't match the metric goal."
  resolution: "Ensure: 1) Entity ID in the event matches the entity ID used in EvaluateFeature, 2) The event data structure matches the metric goal's entityIdKey and valueKey, 3) The event type is 'aws.evidently.custom'."

## Safety Ratings

```
safety_ratings:
  - "get-experiment: GREEN — read-only experiment details"
  - "put-project-events: YELLOW — sends metric events that affect experiment results"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "get-experiment-results: GREEN — read-only results retrieval"
```

## Escalation Conditions

- Metric events confirmed sent but never appearing in experiment results
- PutProjectEvents returning throttling errors at expected event volume
- Metric definitions accepted but producing incorrect statistical results
- Event collection failure affecting multiple experiments

## Data Sensitivity

- `aws evidently get-experiment`: LOW — returns experiment configuration
- `aws evidently put-project-events`: MEDIUM — sends metric data that may contain user behavior
- Metric event data: MEDIUM — contains user interaction metrics and entity identifiers

## Prohibited Actions

- NEVER send fabricated metric events to a production experiment
- NEVER modify metric goal definitions on a running experiment
- NEVER delete metric events or experiment data
- NEVER assume PutProjectEvents success means metrics are collected — verify in results

## Phase 3 — Rollback

1. If test events were sent to a production experiment, document the contamination and assess impact
2. If metric goal definitions were changed, the experiment may need to be recreated
3. If PutProjectEvents was misconfigured, fix the event structure and allow time for new data
4. Document the metric definition issue and resolution
5. Verify experiment results reflect correct metric data after fixes

## Output Format

```yaml
root_cause: "metric_definition — <specific_cause>"
evidence:
  - type: metric_error
    content: "<metric goal definition and event structure details>"
severity: HIGH
mitigation:
  immediate: "Fix metric event structure or goal definition"
  long_term: "Implement metric event validation and testing framework"
```

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
