---
title: "A2 — Aggregator Index Issues"
description: "Diagnose aggregator index configuration and data collection issues"
status: active
severity: HIGH
triggers: ["aggregator not working", "cross-region search incomplete", "aggregator index error"]
owner: devops-agent
objective: "Resolve aggregator index issues affecting cross-region resource search"
context: "Only one aggregator per account. Aggregator collects data from all local indexes. Cross-region search depends on aggregator."
---

## Phase 1 — Triage

MUST:
- Check aggregator index status: `aws resource-explorer-2 list-indexes --type AGGREGATOR`
- List all local indexes: `aws resource-explorer-2 list-indexes --type LOCAL`
- Verify aggregator is ACTIVE: `aws resource-explorer-2 get-index --region <aggregator-region>`

SHOULD:
- Compare local index count against expected regions
- Check for regions without indexes

MAY:
- Review CloudTrail for aggregator-related events
- Check if aggregator was recently created (may still be indexing)

## Phase 2 — Remediate

MUST:
- Ensure aggregator index is in ACTIVE state
- Create local indexes in regions missing them
- Wait for data propagation (up to 36 hours for new indexes)

SHOULD:
- Verify all regions with resources have local indexes
- Monitor aggregator health via CloudWatch

MAY:
- Consider moving aggregator to a different region if latency is a concern
- Set up automated index creation for new regions

## Safety Ratings

```
safety_ratings:
  - "resource-explorer-2 list-indexes: GREEN — read-only enumeration"
  - "resource-explorer-2 get-index: GREEN — read-only status check"
  - "resource-explorer-2 create-index --type LOCAL: YELLOW — creates local index"
  - "resource-explorer-2 update-index-type: RED — changes index type, triggers re-indexing"
```

## Escalation Conditions

- Aggregator index stuck in non-ACTIVE state for more than 2 hours
- Cross-region search returns no results despite local indexes being ACTIVE
- Aggregator needs to be moved to a different region (requires deletion and re-creation)
- Data propagation delay exceeds 48 hours

## Data Sensitivity

- `aws resource-explorer-2 list-indexes`: LOW — returns index metadata and regions
- `aws resource-explorer-2 get-index`: LOW — returns index configuration
- Cross-region search results: MEDIUM — reveals resource inventory across regions

## Prohibited Actions

- NEVER delete the aggregator index without planning for 36-hour re-indexing
- NEVER change aggregator index type without understanding the impact on cross-region search
- NEVER assume missing search results mean resources don't exist — check indexing status
- NEVER create a second aggregator (it will fail)

## Phase 3 — Rollback

1. If aggregator was deleted, recreate in the same region: `aws resource-explorer-2 create-index --type AGGREGATOR --region <region>`
2. Plan for 36-hour re-indexing window after aggregator recreation
3. Verify all local indexes are still ACTIVE after aggregator changes
4. Test cross-region search after aggregator reaches ACTIVE state
5. Document the aggregator region and index topology

## Common Issues

- symptoms: "Cross-region search returns results from some regions but not others"
  diagnosis: "Missing local indexes in some regions or recent index creation still propagating."
  resolution: "Check list-indexes to identify regions without local indexes. Create missing indexes and wait for ACTIVE status."

## Output Format

```yaml
root_cause: "aggregator_index — <specific_cause>"
evidence:
  - type: aggregator_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix aggregator or local index issues"
  long_term: "Automate index creation across all regions"
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
