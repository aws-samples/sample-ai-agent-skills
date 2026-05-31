---
title: "D1 — Cross-Region Indexing"
description: "Diagnose cross-region indexing and search issues"
status: active
severity: HIGH
triggers: ["cross-region search not working", "resources from other regions missing", "multi-region search"]
owner: devops-agent
objective: "Resolve cross-region resource indexing and search issues"
context: "Cross-region search requires an aggregator index plus local indexes in each source region."
---

## Phase 1 — Triage

MUST:
- Verify aggregator index exists: `aws resource-explorer-2 list-indexes --type AGGREGATOR`
- List all local indexes: `aws resource-explorer-2 list-indexes --type LOCAL`
- Check aggregator status: `aws resource-explorer-2 get-index --region <aggregator-region>`

SHOULD:
- Identify regions without local indexes
- Verify all indexes are in ACTIVE state

MAY:
- Test search from the aggregator region: `aws resource-explorer-2 search --query-string "region:<target-region>" --region <aggregator-region>`
- Check data propagation timing for recently created indexes

## Phase 2 — Remediate

MUST:
- Create local indexes in regions missing them: `aws resource-explorer-2 create-index --type LOCAL --region <region>`
- Ensure aggregator index is ACTIVE
- Wait for data propagation (up to 36 hours for new indexes)

SHOULD:
- Deploy local indexes to all active regions using IaC
- Monitor index status across all regions

MAY:
- Set up CloudWatch alarms for index health
- Create a dashboard showing index coverage across regions

## Safety Ratings

```
safety_ratings:
  - "resource-explorer-2 list-indexes: GREEN — read-only enumeration"
  - "resource-explorer-2 get-index: GREEN — read-only status check"
  - "resource-explorer-2 create-index --type LOCAL: YELLOW — creates local index"
  - "resource-explorer-2 search: GREEN — read-only search"
```

## Escalation Conditions

- Aggregator index is not collecting data from specific regions
- Local indexes are ACTIVE but resources don't appear in cross-region search
- Data propagation exceeds 48 hours
- New regions need indexes but automation is not in place

## Data Sensitivity

- `aws resource-explorer-2 list-indexes`: LOW — returns index metadata
- Cross-region search results: MEDIUM — reveals resource inventory across regions
- Region topology: LOW — reveals which regions are in use

## Prohibited Actions

- NEVER delete the aggregator index to troubleshoot cross-region issues
- NEVER assume cross-region search works without verifying both aggregator and local indexes
- NEVER create a second aggregator (only one per account)
- NEVER skip waiting for index ACTIVE status before concluding search is broken

## Phase 3 — Rollback

1. If a local index was created in the wrong region, delete it: `aws resource-explorer-2 delete-index --arn <arn> --type LOCAL`
2. Verify remaining indexes are unaffected: `aws resource-explorer-2 list-indexes`
3. If aggregator was modified, verify cross-region search still works
4. Document the complete index topology
5. Test cross-region search from the aggregator region

## Common Issues

- symptoms: "Search from us-east-1 doesn't show resources in eu-west-1"
  diagnosis: "Missing local index in eu-west-1 or aggregator not in us-east-1."
  resolution: "Create local index in eu-west-1. Ensure aggregator is in us-east-1 (or search from the aggregator region)."

## Output Format

```yaml
root_cause: "cross_region_indexing — <specific_cause>"
evidence:
  - type: indexing_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Create missing local indexes"
  long_term: "Automate index deployment across all regions"
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
