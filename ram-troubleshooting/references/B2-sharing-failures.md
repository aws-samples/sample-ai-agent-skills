---
title: "B2 — Resource Sharing Failures"
description: "Diagnose why resource sharing through RAM is failing"
status: active
severity: HIGH
triggers:
  - "sharing failed"
  - "resource not shared"
  - "AssociateResourceShare error"
  - "share association failed"
owner: devops-agent
objective: "Identify and resolve resource sharing failures in RAM"
context: "Resource sharing can fail when adding resources or principals to a share. Common causes include resource already shared, invalid resource ARN, resource in wrong state, permission issues, or resource-specific constraints."
---

## Phase 1 — Triage

MUST:
- Check resource share status: `aws ram get-resource-shares --resource-share-arns <share-arn>`
- Check resource associations: `aws ram get-resource-share-associations --association-type RESOURCE --resource-share-arns <share-arn>`
- Verify resource ARN is valid and resource exists
- Check CloudTrail for association failures: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AssociateResourceShare --max-results 10`

SHOULD:
- Verify the resource is not in another share
- Check resource state (must be active/available)
- Verify IAM permissions for the resource service
- Check for resource-specific sharing constraints

MAY:
- Verify the resource owner matches the share owner
- Check for service-specific quotas

## Phase 2 — Remediate

MUST:
- Ensure resource ARN is correct and resource exists
- Verify resource is in a shareable state
- Fix IAM permissions for both RAM and the resource service

SHOULD:
- Remove resource from conflicting shares if needed
- Verify resource-specific prerequisites
- Test sharing with a single resource first

MAY:
- Automate resource sharing validation
- Create pre-sharing checklists for each resource type

## Common Issues

- symptoms: "Resource already associated with another share"
  diagnosis: "Some resource types can only be in one share at a time."
  resolution: "Remove resource from existing share or add principals to the existing share."

- symptoms: "Association status FAILED"
  diagnosis: "Resource-specific constraint or permission issue."
  resolution: "Check CloudTrail for detailed error. Verify resource-specific prerequisites."

## Output Format

```yaml
root_cause: "sharing_failure — <specific_cause>"
evidence:
  - type: share_status
    content: "<resource share status>"
  - type: association_status
    content: "<resource association details>"
severity: HIGH
mitigation:
  immediate: "Fix the specific sharing blocker"
  long_term: "Automate sharing validation and create pre-sharing checklists"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Resource share configurations may contain account IDs and organizational structure
- Principal ARNs reveal cross-account relationships
- Permission policies may expose internal access patterns
- Classification: INTERNAL - contains organizational topology data

## Prohibited Actions

- NEVER modify or delete resource shares in production without owner approval
- NEVER change RAM permissions that could revoke cross-account access to shared resources
- NEVER disable organization sharing without coordinating with all member accounts
- NEVER remove principals from active resource shares without impact analysis

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
