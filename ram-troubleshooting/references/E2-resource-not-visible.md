---
title: "E2 — Resource Not Visible"
description: "Diagnose why shared resources are not visible in participant accounts"
status: active
severity: MEDIUM
triggers:
  - "resource not visible"
  - "shared resource not showing"
  - "cannot find shared resource"
  - "resource not appearing"
owner: devops-agent
objective: "Identify why shared resources are not visible and resolve visibility issues"
context: "Shared resources may not be visible due to invitation not accepted, organization sharing not enabled, resource share not active, wrong region, or service-specific visibility requirements. Some shared resources appear in service-specific consoles, not the RAM console."
---

## Phase 1 — Triage

MUST:
- Check resource shares available to the account: `aws ram get-resource-shares --resource-owner OTHER-ACCOUNTS`
- List shared resources: `aws ram list-resources --resource-owner OTHER-ACCOUNTS`
- Verify invitation acceptance (for external shares): `aws ram get-resource-share-invitations`
- Check the correct region (shared resources are region-specific)

SHOULD:
- Verify organization sharing is enabled (for org shares)
- Check the service-specific console for the shared resource
- Verify the account is a principal in the resource share
- Check resource share status is ACTIVE

MAY:
- Check CloudTrail for resource access attempts
- Verify no SCPs are hiding resources

## Phase 2 — Remediate

MUST:
- Accept pending invitations for external shares
- Verify organization sharing is enabled for org shares
- Check the correct region for the shared resource

SHOULD:
- Use service-specific APIs to find shared resources (e.g., describe-subnets for shared subnets)
- Verify the resource share is ACTIVE
- Document where shared resources appear in each service console

MAY:
- Create resource discovery automation
- Set up dashboards showing shared resources

## Common Issues

- symptoms: "No resource shares visible in participant account"
  diagnosis: "Invitation not accepted, org sharing not enabled, or wrong region."
  resolution: "Accept invitation, enable org sharing, or switch to the correct region."

- symptoms: "Resource share visible but resources not in service console"
  diagnosis: "Some services require specific API calls to list shared resources."
  resolution: "Use service-specific APIs (e.g., aws ec2 describe-subnets for shared subnets)."

## Output Format

```yaml
root_cause: "resource_not_visible — <specific_cause>"
evidence:
  - type: share_status
    content: "<resource share visibility>"
  - type: invitation_status
    content: "<invitation acceptance status>"
severity: MEDIUM
mitigation:
  immediate: "Fix visibility issue (accept invitation, enable org sharing, check region)"
  long_term: "Document shared resource discovery procedures"
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
