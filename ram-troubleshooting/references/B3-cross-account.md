---
title: "B3 — Cross-Account Resource Access"
description: "Diagnose cross-account resource access issues through RAM"
status: active
severity: HIGH
triggers:
  - "cross-account access failed"
  - "cannot access shared resource"
  - "shared resource not working"
  - "cross-account permission denied"
owner: devops-agent
objective: "Identify and resolve cross-account resource access issues through RAM"
context: "Cross-account access through RAM requires the resource share to be active, the principal to be associated, the invitation accepted (if external), and both RAM permissions and IAM policies to allow the action. Resource-specific constraints may also apply."
---

## Phase 1 — Triage

MUST:
- Verify resource share is ACTIVE: `aws ram get-resource-shares --resource-owner OTHER-ACCOUNTS`
- Check principal association: `aws ram get-resource-share-associations --association-type PRINCIPAL --resource-share-arns <share-arn>`
- Verify RAM permission on the share: `aws ram list-permissions --resource-type <type>`
- Check IAM permissions in the participant account

SHOULD:
- Verify invitation was accepted (for external shares)
- Check resource-specific access requirements
- Test with a simple operation on the shared resource
- Verify the resource is in the same region as the participant

MAY:
- Check CloudTrail in both accounts for access attempts
- Verify no SCPs are blocking access

## Phase 2 — Remediate

MUST:
- Ensure resource share is ACTIVE and principal is ASSOCIATED
- Verify both RAM permission and IAM policy allow the action
- Accept invitation if pending (external shares)

SHOULD:
- Test access with minimal permissions first
- Document cross-account access requirements
- Verify resource-specific constraints (e.g., shared subnet limitations)

MAY:
- Set up cross-account monitoring
- Create access verification automation

## Common Issues

- symptoms: "Access denied when using shared resource"
  diagnosis: "IAM policy in participant account doesn't allow the action."
  resolution: "Add IAM permissions for the shared resource actions in the participant account."

- symptoms: "Shared resource not visible in participant account"
  diagnosis: "Invitation not accepted or organization sharing not enabled."
  resolution: "Accept invitation or enable organization sharing. Check resource share status."

## Output Format

```yaml
root_cause: "cross_account_access — <specific_cause>"
evidence:
  - type: share_status
    content: "<resource share and association status>"
  - type: permissions
    content: "<RAM permission and IAM policy evaluation>"
severity: HIGH
mitigation:
  immediate: "Fix permissions and access configuration"
  long_term: "Document cross-account access patterns and automate verification"
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
