---
title: "E2 — Tag Management"
description: "Diagnose tagging issues on Outposts Server resources"
status: active
severity: LOW
triggers: ["tags not applied to outpost", "missing tags on outpost resources", "tag policy violation", "cost allocation tags"]
owner: devops-agent
objective: "Resolve tagging issues for Outposts Server resources for cost tracking and governance"
context: "Outposts resources (Outpost, site, instances, volumes) support AWS tags. Tags are critical for cost allocation, governance, and automation. Tag policies from AWS Organizations may enforce requirements."
---

## Phase 1 — Triage

MUST:
- List tags on Outpost: `aws outposts list-tags-for-resource --resource-arn <outpost-arn>`
- Check instance tags: `aws ec2 describe-tags --filters "Name=resource-id,Values=<instance-id>" --query 'Tags[*].{Key:Key,Value:Value}'`
- Verify tag policies: `aws organizations describe-effective-policy --policy-type TAG_POLICY --target-id <account-id> --query 'EffectivePolicy.PolicyContent'`

SHOULD:
- Check if cost allocation tags are activated in the billing console
- Verify tag-based IAM policies are not blocking operations
- Review tag compliance across all Outpost resources

MAY:
- Implement AWS Config rules for tag compliance
- Set up automated tagging via Lambda or EventBridge

## Phase 2 — Remediate

MUST:
- Apply required tags to Outpost resources: `aws outposts tag-resource --resource-arn <arn> --tags Key=Environment,Value=Production`
- Tag EC2 instances and EBS volumes on the Outpost
- Ensure tags comply with organizational tag policies

SHOULD:
- Implement tag-on-create enforcement via IAM policies
- Activate cost allocation tags for Outpost cost tracking

MAY:
- Deploy AWS Config rules for ongoing tag compliance monitoring
- Create automated remediation for missing tags

## Common Issues

- symptoms: "Tags applied via API but not showing in cost reports"
  diagnosis: "Cost allocation tags not activated in the billing console."
  resolution: "Activate cost allocation tags in AWS Billing console > Cost allocation tags. Tags take 24 hours to appear in reports."

## Safety Ratings

```
safety_ratings:
  - "list-tags-for-resource: GREEN — read-only tag enumeration"
  - "describe-tags: GREEN — read-only tag check"
  - "tag-resource: GREEN — adds metadata, non-destructive"
  - "untag-resource: YELLOW — removes metadata, may affect automation or policies"
```

## Escalation Conditions

- Tag policy violations blocking critical resource operations
- Cost allocation tags not appearing after 48 hours of activation
- Tag-based IAM policies causing unexpected access denials
- Organization-wide tag policy changes affecting Outpost operations

## Data Sensitivity

- `aws outposts list-tags-for-resource`: LOW — returns tag key-value pairs
- `aws ec2 describe-tags`: LOW — returns resource tags
- Tag values: MEDIUM — may contain environment, cost center, or project identifiers
- Tag policies: LOW — returns governance rules

## Prohibited Actions

- NEVER remove tags without understanding their use in IAM policies, automation, or cost allocation
- NEVER apply tags that violate organizational tag policies
- NEVER use tags to store sensitive information (passwords, keys, PII)
- NEVER modify tags on resources owned by other teams without coordination

## Phase 3 — Rollback

1. If tags were removed in error, re-apply them: `aws outposts tag-resource --resource-arn <arn> --tags Key=<key>,Value=<value>`
2. If incorrect tags were applied, update with correct values
3. If tag policies were modified, revert the policy changes in AWS Organizations
4. Verify tag-dependent automation and IAM policies still function correctly
5. Document tag changes for audit trail

## Output Format

```yaml
root_cause: "tag_management — <specific_cause>"
evidence:
  - type: tag_status
    content: "<tag configuration and compliance details>"
severity: LOW
mitigation:
  immediate: "Apply or correct tags"
  long_term: "Implement automated tag enforcement"
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
