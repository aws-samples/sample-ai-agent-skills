---
title: "D1 — Segment Configuration"
description: "Diagnose core network segment creation, isolation, and configuration issues"
status: active
severity: HIGH
triggers: ["segment error", "segment not created", "segment isolation broken", "segment configuration failed"]
owner: devops-agent
objective: "Resolve core network segment configuration and isolation issues"
context: "Segments provide network isolation within a core network. Each segment has its own route table and attachments are assigned to segments via attachment-policies."
---

## Phase 1 — Triage

MUST:
- Get core network details including segments: `aws networkmanager get-core-network --core-network-id <id> --query 'CoreNetwork.Segments'`
- Get the active policy to review segment definitions: `aws networkmanager get-core-network-policy --core-network-id <id>`
- List attachments to verify segment assignments: `aws networkmanager list-attachments --core-network-id <id>`

SHOULD:
- Verify segment edge-locations match the intended regions
- Check segment-actions for unintended share or route-between rules
- Verify require-attachment-acceptance setting per segment

MAY:
- Review segment route tables for unexpected routes
- Check if isolate-attachments is set correctly for the segment

## Phase 2 — Remediate

MUST:
- Fix segment definitions in the core network policy
- Ensure each segment has the correct edge-locations
- Verify isolate-attachments setting (true = attachments in same segment cannot communicate)

SHOULD:
- Submit corrected policy: `aws networkmanager put-core-network-policy --core-network-id <id> --policy-document file://policy.json`
- Review and execute the change set

MAY:
- Implement segment naming conventions for clarity
- Document segment purpose and allowed traffic patterns

## Common Issues

- symptoms: "Attachments in the same segment cannot communicate"
  diagnosis: "isolate-attachments is set to true for the segment."
  resolution: "Set isolate-attachments to false in the segment definition if inter-attachment communication is required."

- symptoms: "Segment not visible in specific region"
  diagnosis: "Segment edge-locations do not include the target region."
  resolution: "Add the region to the segment's edge-locations array in the core network policy."

## Safety Ratings

```
safety_ratings:
  - "get-core-network (segments): GREEN — read-only segment information"
  - "get-core-network-policy: GREEN — read-only policy retrieval"
  - "put-core-network-policy: YELLOW — creates new policy version"
  - "execute-core-network-change-set: RED — activates segment changes affecting isolation"
```

## Escalation Conditions

- Segment isolation breach allowing unintended cross-segment traffic
- Segment deletion in policy would orphan existing attachments
- Segment configuration changes affect compliance-mandated network isolation
- Multiple segments with overlapping purposes causing architectural confusion

## Data Sensitivity

- `aws networkmanager get-core-network`: MEDIUM — reveals segment topology
- `aws networkmanager get-core-network-policy`: MEDIUM — reveals segmentation strategy
- `aws networkmanager list-attachments`: LOW — returns attachment metadata

## Prohibited Actions

- NEVER remove a segment from the policy if attachments are assigned to it
- NEVER disable segment isolation without security team approval
- NEVER rename segments in production — this can orphan attachments
- NEVER add share or route-between actions to compliance-sensitive segments without review

## Phase 3 — Rollback

1. Create a rollback policy with the previous segment configuration
2. Submit and execute the rollback policy
3. Verify segment isolation is restored: check routes in each segment
4. Confirm attachments are still assigned to correct segments
5. Document the rollback and notify the network and security teams

## Output Format

```yaml
root_cause: "segment_configuration — <specific_cause>"
evidence:
  - type: segment_state
    content: "<segment details>"
severity: HIGH
mitigation:
  immediate: "Fix segment configuration"
  long_term: "Implement segment governance policies"
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
