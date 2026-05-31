---
title: "B2 — Missing Resources"
description: "Diagnose resources missing from Resource Explorer search results"
status: active
severity: HIGH
triggers: ["resource not found", "missing resource", "resource not showing", "search returns empty"]
owner: devops-agent
objective: "Identify why resources are not appearing in Resource Explorer search results"
context: "Missing resources can be caused by indexing delays, unsupported types, missing indexes, or view filters."
---

## Phase 1 — Triage

MUST:
- Verify the resource exists: use service-specific API (e.g., `aws ec2 describe-instances --instance-ids <id>`)
- Check if the resource type is supported: `aws resource-explorer-2 list-supported-resource-types --query "ResourceTypes[?ResourceType=='<type>']"`
- Verify an index exists in the resource's region: `aws resource-explorer-2 get-index --region <resource-region>`

SHOULD:
- Check the view for filters that may exclude the resource: `aws resource-explorer-2 get-view --view-arn <view-arn>`
- Verify index status is ACTIVE (not CREATING)

MAY:
- Check when the resource was created (indexing delay up to 36 hours)
- Search with the default view to rule out view filter issues

## Phase 2 — Remediate

MUST:
- Create a local index in the resource's region if missing
- Wait for indexing if the resource was recently created (up to 36 hours)
- Use the default view to confirm the resource is indexed

SHOULD:
- Update custom views to include the resource type if filtered out
- Verify the aggregator index is collecting from the resource's region

MAY:
- Use AWS Config for resource types not supported by Resource Explorer
- Set up notifications for index status changes

## Safety Ratings

```
safety_ratings:
  - "service-specific describe: GREEN — read-only resource verification"
  - "resource-explorer-2 list-supported-resource-types: GREEN — read-only type list"
  - "resource-explorer-2 get-index: GREEN — read-only index status"
  - "resource-explorer-2 get-view: GREEN — read-only view details"
```

## Escalation Conditions

- Resource type is supported but resource does not appear after 48 hours
- Multiple resources across different types are missing
- Index is ACTIVE but search returns no results for any resource type
- Missing resources affect compliance or security monitoring

## Data Sensitivity

- Service-specific describe APIs: MEDIUM — reveals resource configuration details
- `aws resource-explorer-2 search`: MEDIUM — reveals resource inventory
- Resource ARNs and tags: MEDIUM — may contain environment or ownership information

## Prohibited Actions

- NEVER conclude a resource doesn't exist based solely on Resource Explorer search results
- NEVER delete and recreate indexes to fix missing resources without exhausting other options
- NEVER modify resource tags to make them appear in filtered views without approval
- NEVER assume all resource types are supported by Resource Explorer

## Phase 3 — Rollback

1. If a view was modified to troubleshoot, restore original filters
2. If an index was created in a new region, verify it reaches ACTIVE state
3. Document which resource types are not supported for future reference
4. Verify the missing resource appears after remediation steps
5. Update team documentation on Resource Explorer coverage gaps

## Common Issues

- symptoms: "EC2 instances exist but don't appear in Resource Explorer search"
  diagnosis: "No local index in the instance's region, or index is still in CREATING state."
  resolution: "Create a local index in the region: aws resource-explorer-2 create-index --type LOCAL --region <region>. Wait for ACTIVE status."

## Output Format

```yaml
root_cause: "missing_resource — <specific_cause>"
evidence:
  - type: resource_check
    content: "<verification details>"
severity: HIGH
mitigation:
  immediate: "Fix indexing or view configuration"
  long_term: "Ensure all regions have local indexes"
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
