---
title: "E1 — Segment Configuration"
description: "Diagnose segment creation and matching issues"
status: active
severity: MEDIUM
triggers: ["segment not matching", "segment creation failed", "entity not in segment", "CreateSegment error"]
owner: devops-agent
objective: "Resolve segment configuration and entity matching issues"
context: "Segments define user subsets using pattern matching rules on entity attributes. Attributes must be passed in EvaluateFeature calls for matching."
---

## Phase 1 — Triage

MUST:
- List segments: `aws evidently list-segments`
- Get segment details: `aws evidently get-segment --segment <segment-name>`
- Review the segment pattern (rule definition) for correctness

SHOULD:
- Verify the application passes required entity attributes in EvaluateFeature calls
- Check if the segment is referenced by any active launches or experiments
- Test segment matching with a known entity

MAY:
- Review CloudTrail for CreateSegment or UpdateSegment errors
- Check if multiple segments have conflicting rules

## Phase 2 — Remediate

MUST:
- Fix segment pattern to match the intended entity attributes
- Ensure the application passes the required attributes in EvaluateFeature calls
- Verify the segment pattern syntax is valid JSON

SHOULD:
- Test segment matching with representative entity attributes
- Document the expected entity attributes for each segment

MAY:
- Create test segments for validation before production use
- Set up monitoring for segment match rates

## Common Issues

- symptoms: "Segment exists but entities are not matching the segment rules"
  diagnosis: "Entity attributes may not be passed in EvaluateFeature calls, or attribute names don't match the segment pattern."
  resolution: "Verify the application passes entity attributes: `aws evidently evaluate-feature --project <name> --feature <feature-name> --entity-id <id> --evaluation-context '{\"key\":\"value\"}'`. Ensure attribute names in the context match the segment pattern exactly."

## Safety Ratings

```
safety_ratings:
  - "list-segments: GREEN — read-only segment enumeration"
  - "get-segment: GREEN — read-only segment details"
  - "CreateSegment: YELLOW — creates a new segment resource"
  - "delete-segment: RED — removes segment, may affect launches/experiments"
```

## Escalation Conditions

- Segment matching produces incorrect results affecting production traffic
- Segment pattern changes needed during an active launch or experiment
- Multiple segments with overlapping rules causing confusion
- Segment deletion required but referenced by active resources

## Data Sensitivity

- `aws evidently list-segments`: LOW — returns segment metadata
- `aws evidently get-segment`: MEDIUM — reveals targeting rules and user segmentation strategy
- Entity attributes: MEDIUM — may contain user demographic or behavioral data

## Prohibited Actions

- NEVER delete a segment that is referenced by active launches or experiments
- NEVER modify segment patterns without understanding the impact on active traffic splits
- NEVER create segments with overly broad patterns that match all entities
- NEVER assume entity attributes are passed automatically — they must be explicitly included

## Phase 3 — Rollback

1. If a segment was modified incorrectly, update the pattern: `aws evidently update-segment --segment <name> --pattern <previous-pattern>`
2. If a segment was deleted, recreate it with the original pattern
3. Verify launches and experiments using the segment are functioning correctly
4. Document the segment configuration change and its impact
5. Confirm entity matching returns to expected behavior

## Output Format

```yaml
root_cause: "segment_configuration — <specific_cause>"
evidence:
  - type: segment_issue
    content: "<segment pattern and matching details>"
severity: MEDIUM
mitigation:
  immediate: "Fix segment pattern or entity attribute passing"
  long_term: "Implement segment testing in CI/CD pipeline"
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
