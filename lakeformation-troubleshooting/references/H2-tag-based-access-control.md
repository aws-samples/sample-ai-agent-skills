---
title: "H2 — Tag-Based Access Control (LF-Tags)"
description: "Diagnose LF-Tag-based access control issues"
status: active
severity: MEDIUM
triggers:
  - "LF-Tag error"
  - "tag-based access not working"
  - "TBAC issue"
  - "LF-Tag permission"
owner: devops-agent
objective: "Identify and resolve LF-Tag-based access control issues"
context: "LF-Tags provide attribute-based access control for Lake Formation. Tags are key-value pairs assigned to databases, tables, and columns. Tag-based permissions grant access based on tag values. Tags inherit down the resource hierarchy. This simplifies permission management at scale."
---

## Phase 1 — Triage

MUST:
- List LF-Tags: `aws lakeformation list-lf-tags`
- Check tags on resource: `aws lakeformation get-resource-lf-tags --resource '{"Table":{"DatabaseName":"<db>","Name":"<table>"}}'`
- Check tag-based permissions: `aws lakeformation list-permissions --resource-type TAG`
- Search tables by tag: `aws lakeformation search-tables-by-lf-tags --expression '[{"TagKey":"<key>","TagValues":["<value>"]}]'`

SHOULD:
- Verify tag inheritance from database to table
- Check for explicit tag overrides on child resources
- Verify tag values are case-sensitive matches
- Check if the principal has tag-based grants matching the resource tags

MAY:
- Review tag governance policies
- Check for tag conflicts across resources

## Phase 2 — Remediate

MUST:
- Ensure LF-Tags are defined before assigning
- Assign tags to resources at the appropriate level
- Create tag-based permission grants matching tag values

SHOULD:
- Use tag inheritance for consistent access patterns
- Document tag taxonomy and permission mappings
- Test tag-based access with sample queries

MAY:
- Automate tag assignment for new resources
- Create tag governance dashboards

## Common Issues

- symptoms: "Tag-based permission not granting access"
  diagnosis: "Tag values don't match between grant and resource, or tag not assigned."
  resolution: "Verify tag assignment on resource. Ensure grant tag values match exactly (case-sensitive)."

- symptoms: "Unexpected access through tag inheritance"
  diagnosis: "Database tag inherited by all tables, granting broader access than intended."
  resolution: "Override inherited tags on specific tables. Use explicit tags for fine-grained control."

## Output Format

```yaml
root_cause: "tag_based_access — <specific_cause>"
evidence:
  - type: lf_tags
    content: "<LF-Tag assignments on resources>"
  - type: tag_permissions
    content: "<tag-based permission grants>"
severity: MEDIUM
mitigation:
  immediate: "Fix tag assignments or tag-based permissions"
  long_term: "Implement tag governance and automate tag management"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** revoke Lake Formation permissions on production databases without explicit customer approval
- **NEVER** modify IAMAllowedPrincipals settings without understanding the full permission inheritance chain
- **NEVER** deregister S3 data locations that are actively referenced by Glue tables or crawlers
- **NEVER** disable hybrid access mode on databases with active IAM-based consumers
- **NEVER** alter cross-account resource shares (RAM) without coordinating with all participating accounts

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
