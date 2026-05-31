---
title: "C2 — Auto-Sharing Configuration"
description: "Diagnose RAM auto-sharing and automatic resource distribution issues"
status: active
severity: MEDIUM
triggers:
  - "auto-sharing not working"
  - "resources not automatically shared"
  - "new account not getting shared resources"
  - "auto-share configuration"
owner: devops-agent
objective: "Identify and resolve RAM auto-sharing configuration issues"
context: "When organization sharing is enabled and resources are shared with an organization or OU principal, new accounts added to the organization/OU automatically get access. Auto-sharing depends on proper organization sharing setup and correct principal scope."
---

## Phase 1 — Triage

MUST:
- Verify organization sharing is enabled
- Check resource share principals include org/OU ARNs: `aws ram get-resource-share-associations --association-type PRINCIPAL --resource-share-arns <share-arn>`
- Verify new account is in the correct OU: `aws organizations list-parents --child-id <account-id>`
- Check resource share status is ACTIVE

SHOULD:
- Verify the new account is active in the organization
- Check if the account was recently added (propagation delay)
- Review the resource share scope configuration
- Check for SCPs affecting the new account

MAY:
- Test resource access from the new account
- Check CloudTrail for sharing events related to the new account

## Phase 2 — Remediate

MUST:
- Ensure organization sharing is enabled
- Use organization or OU ARN as principal (not individual account IDs)
- Verify new accounts are in the correct OU

SHOULD:
- Wait for propagation after adding new accounts (can take minutes)
- Test access from new accounts after setup
- Document auto-sharing behavior for onboarding

MAY:
- Create account onboarding automation that verifies RAM access
- Set up monitoring for new account resource access

## Common Issues

- symptoms: "New account doesn't see shared resources"
  diagnosis: "Account not in the OU targeted by the resource share, or propagation delay."
  resolution: "Verify account OU membership. Wait for propagation. Check resource share principals."

- symptoms: "Auto-sharing works for some resources but not others"
  diagnosis: "Different resource shares may target different OUs."
  resolution: "Check each resource share's principal configuration."

## Output Format

```yaml
root_cause: "auto_sharing — <specific_cause>"
evidence:
  - type: share_principals
    content: "<resource share principal configuration>"
  - type: account_ou
    content: "<account OU membership>"
severity: MEDIUM
mitigation:
  immediate: "Fix principal scope or account OU membership"
  long_term: "Standardize auto-sharing patterns and create onboarding verification"
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
