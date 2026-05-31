---
title: "B3 — Rule Group Versioning Issues"
description: "Diagnose problems caused by managed rule group version changes"
status: active
severity: MEDIUM
triggers:
  - "rule group version"
  - "version change"
  - "rules changed"
  - "unexpected blocking after update"
  - "rule group update"
owner: devops-agent
objective: "Identify and manage rule group version changes that affect WAF behavior"
context: "AWS managed rule groups are versioned. By default, web ACLs use the default version set by AWS. When AWS updates the default version, behavior may change. You can pin to a specific version to prevent automatic updates. Expired versions are automatically upgraded to the current default. SNS notifications are available for version changes."
---

## Phase 1 — Triage

MUST:
- Check the current version in use: `aws wafv2 get-web-acl --name <acl-name> --scope <scope> --id <acl-id> --query 'WebACL.Rules[?Statement.ManagedRuleGroupStatement].Statement.ManagedRuleGroupStatement.Version'`
- List available versions: `aws wafv2 describe-managed-rule-group --vendor-name AWS --name <rule-group-name> --scope <scope> --query '{CurrentDefaultVersion:VersionName,AvailableVersions:AvailableLabels}'`
- Check if the version is pinned or using the default: a null Version field means using the default
- Review CloudTrail for recent web ACL updates: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=UpdateWebACL --start-time <start> --end-time <end>`

SHOULD:
- Compare the blocking behavior before and after the version change using CloudWatch metrics
- Check the SNS topic for managed rule group notifications
- Review the version release notes from AWS

MAY:
- Test the new version in a staging web ACL before applying to production
- Compare rule lists between versions using describe-managed-rule-group with version parameter

## Phase 2 — Remediate

MUST:
- Pin to a specific version if the new default version causes issues: update the web ACL with the Version field set to the desired version
- If pinned to an expiring version, test and migrate to a newer version before expiration
- Monitor for version expiration notifications

SHOULD:
- Subscribe to the SNS topic for managed rule group version changes
- Implement a version testing workflow: new version in Count mode → validate → switch to Block
- Document the current pinned versions for all managed rule groups

MAY:
- Automate version testing using AWS Lambda triggered by SNS notifications
- Create a rollback procedure for quick version reversion

## Common Issues

- symptoms: "New false positives appeared without any web ACL changes"
  diagnosis: "AWS updated the default version of a managed rule group, adding or modifying rules."
  resolution: "Pin to the previous version. Test the new version in Count mode."

- symptoms: "Pinned version stopped working"
  diagnosis: "The pinned version expired and was automatically upgraded to the current default."
  resolution: "Pin to a current supported version. Monitor expiration dates."

- symptoms: "Cannot determine which version is in use"
  diagnosis: "Version field is null, meaning the web ACL uses the default version set by AWS."
  resolution: "Use describe-managed-rule-group to see the current default version."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Pin to specific version | YELLOW | Version lock; reversible |
| Subscribe to SNS notifications | GREEN | Monitoring only; no rule change |
| Test new version in Count mode | YELLOW | Adds Count override; reversible |

## Escalation Conditions

- Production web ACL rule changes
- Shield Advanced configuration changes
- Version changes affecting production traffic blocking

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-web-acl` (version info) | LOW | Version configuration |
| `describe-managed-rule-group` | LOW | Version metadata |
| CloudTrail events | LOW | API call history |

## Prohibited Actions

- NEVER suggest removing all WAF rules to fix false positives
- NEVER suggest disabling Shield Advanced
- NEVER suggest setting rate limit to maximum to "disable" rate limiting

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Version pin | Unpin or pin to previous version via `update-web-acl` |
| Count mode testing | Remove Count override via `update-web-acl` |

## Output Format

```yaml
root_cause: "rule_group_versioning — <specific_cause>"
evidence:
  - type: current_version
    content: "<version in use>"
  - type: default_version
    content: "<AWS default version>"
  - type: version_change_date
    content: "<when the version changed>"
severity: MEDIUM
mitigation:
  immediate: "Pin to a known-good version"
  long_term: "Implement version testing workflow and SNS monitoring"
```

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
  - "NEVER suggest disabling WAF to fix access issues"
  - "NEVER suggest removing all WAF rules"
  - "NEVER suggest allowing all IPs to bypass rate limiting"
