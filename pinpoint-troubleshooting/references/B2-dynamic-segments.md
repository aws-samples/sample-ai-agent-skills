---
title: "B2 — Dynamic Segment Issues"
description: "Diagnose dynamic segment computation and membership problems"
status: active
severity: MEDIUM
triggers:
  - "dynamic segment wrong size"
  - "segment not updating"
  - "segment filter not working"
  - "endpoints missing from segment"
owner: devops-agent
objective: "Identify and resolve dynamic segment issues"
context: "Dynamic segments compute membership based on endpoint attributes. They are eventually consistent and may not reflect recent changes immediately."
---

## Phase 1 — Triage

MUST:
- Get segment definition: `aws pinpoint get-segment --application-id <app-id> --segment-id <seg-id>`
- Check segment version: `aws pinpoint get-segment-versions --application-id <app-id> --segment-id <seg-id>`
- Verify sample endpoint attributes: `aws pinpoint get-endpoint --application-id <app-id> --endpoint-id <endpoint-id>`
- Check application endpoint count: `aws pinpoint get-application-settings --application-id <app-id>`

SHOULD:
- Verify filter attribute names match endpoint attributes exactly (case-sensitive)
- Check if endpoints have the expected channel type
- Test with simplified filter criteria

MAY:
- Export segment to verify membership
- Check for stale endpoints (inactive > 30 days)

## Phase 2 — Remediate

MUST:
- Fix attribute name mismatches in segment filters
- Ensure endpoints have required attributes populated
- Allow propagation time after endpoint updates

SHOULD:
- Use consistent attribute naming conventions
- Clean up stale endpoints regularly

MAY:
- Implement segment size monitoring and alerting

## Common Issues

- symptoms: "Segment size doesn't match expected count"
  diagnosis: "Attribute name case mismatch or endpoints lack the filtered attribute."
  resolution: "Verify attribute names are case-sensitive matches. Check endpoint attributes."

- symptoms: "Recently added endpoints not in segment"
  diagnosis: "Eventual consistency delay."
  resolution: "Wait for propagation (minutes). Re-check segment size."

## Output Format

```yaml
root_cause: "dynamic_segment — <specific_cause>"
evidence:
  - type: segment_definition
    content: "<segment filter criteria>"
  - type: endpoint_attributes
    content: "<sample endpoint attributes>"
severity: MEDIUM
mitigation:
  immediate: "Fix filter criteria or wait for propagation"
  long_term: "Standardize endpoint attribute naming"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: HIGH — dynamic segments query customer attributes
- Service Disruption Risk: LOW — segment issues affect targeting accuracy
- Compliance Impact: HIGH — dynamic segmentation must comply with data protection regulations

## Escalation Conditions
- Dynamic segment criteria produce incorrect results
- Segment evaluation performance degrades
- Segment includes opted-out customers
- Segment criteria changes affect active campaigns
- Dynamic segment size fluctuates unexpectedly

## Data Sensitivity
- Customer attributes — RESTRICTED
- Segment criteria expressions — CONFIDENTIAL
- Segment evaluation results — INTERNAL
- Customer behavioral data — RESTRICTED
- Never expose individual customer data from segment queries

## Prohibited Actions
- Do NOT create dynamic segments that discriminate based on protected characteristics
- Do NOT modify segment criteria used by active campaigns without approval
- Do NOT include opted-out customers in segments
- Do NOT share segment criteria externally
- Do NOT ignore segment evaluation errors
- Do NOT create segments without data governance review

## Phase 3 — Rollback
- Restore previous segment criteria
- Revert segment configuration changes
- Verify segment membership after changes
- Restore campaign-segment associations
- Document segment issues and data governance findings
- Update dynamic segmentation procedures

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
