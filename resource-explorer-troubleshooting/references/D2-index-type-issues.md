---
title: "D2 — Index Type Issues"
description: "Diagnose index type configuration and conversion issues"
status: active
severity: MEDIUM
triggers: ["wrong index type", "convert index", "local vs aggregator", "index type change"]
owner: devops-agent
objective: "Resolve index type configuration and conversion issues"
context: "Two index types: LOCAL (region-specific) and AGGREGATOR (cross-region). Only one aggregator per account."
---

## Phase 1 — Triage

MUST:
- Check current index type: `aws resource-explorer-2 get-index`
- List all indexes with types: `aws resource-explorer-2 list-indexes --query 'Indexes[*].{Region:Region,Type:Type}'`
- Verify the desired index topology

SHOULD:
- Confirm which region should host the aggregator
- Check if an aggregator already exists in another region

MAY:
- Review the original index creation in CloudTrail
- Assess the impact of changing index types

## Phase 2 — Remediate

MUST:
- To change index type, use update-index-type: `aws resource-explorer-2 update-index-type --arn <arn> --type <LOCAL|AGGREGATOR>`
- Ensure only one aggregator exists after the change
- Wait for the index to reach ACTIVE state after type change

SHOULD:
- Plan for re-indexing delay when changing types
- Notify teams that depend on cross-region search

MAY:
- Use IaC to manage index topology consistently
- Document the rationale for the chosen aggregator region

## Safety Ratings

```
safety_ratings:
  - "resource-explorer-2 get-index: GREEN — read-only index details"
  - "resource-explorer-2 list-indexes: GREEN — read-only enumeration"
  - "resource-explorer-2 update-index-type: RED — changes index type, triggers re-indexing"
```

## Escalation Conditions

- Index type change fails with ConflictException
- Re-indexing after type change takes longer than expected
- Cross-region search breaks during index type transition
- Multiple teams disagree on the aggregator region

## Data Sensitivity

- `aws resource-explorer-2 get-index`: LOW — returns index configuration
- `aws resource-explorer-2 list-indexes`: LOW — returns index metadata
- Index type changes: LOW — affects search topology, not resource data

## Prohibited Actions

- NEVER change the aggregator to LOCAL without understanding cross-region search will break
- NEVER change index types during business hours without a maintenance window
- NEVER create a second aggregator — demote the existing one first
- NEVER assume index type changes are instant — re-indexing takes up to 36 hours

## Phase 3 — Rollback

1. If index type was changed incorrectly, revert: `aws resource-explorer-2 update-index-type --arn <arn> --type <original-type>`
2. Wait for the index to reach ACTIVE state after rollback
3. Verify cross-region search works from the aggregator region
4. Test search results in affected regions
5. Notify teams of the rollback and expected re-indexing delay

## Common Issues

- symptoms: "update-index-type returns ConflictException when changing LOCAL to AGGREGATOR"
  diagnosis: "An aggregator already exists in another region."
  resolution: "Demote the existing aggregator to LOCAL first, then promote the desired region to AGGREGATOR."

## Output Format

```yaml
root_cause: "index_type — <specific_cause>"
evidence:
  - type: type_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix index type configuration"
  long_term: "Document and automate index topology"
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
