---
title: "A2 — Invitation Issues"
description: "Diagnose RAM resource share invitation problems"
status: active
severity: MEDIUM
triggers:
  - "invitation not received"
  - "invitation expired"
  - "pending invitation"
  - "invitation error"
owner: devops-agent
objective: "Identify and resolve RAM invitation issues"
context: "RAM invitations are sent when sharing with accounts outside the organization or when organization sharing is not enabled. Invitations expire after 7 days. Within an organization with RAM sharing enabled, invitations are NOT used — resources are automatically available."
---

## Phase 1 — Triage

MUST:
- Check pending invitations: `aws ram get-resource-share-invitations --query 'resourceShareInvitations[?status==PENDING]'`
- Verify if organization sharing is enabled (invitations not needed within org)
- Check invitation status: `aws ram get-resource-share-invitations`
- Verify the recipient account ID is correct

SHOULD:
- Check if invitation expired (7-day expiration)
- Verify allowExternalPrincipals is true on the share
- Check CloudTrail for invitation events
- Verify the recipient account can receive invitations (no SCP blocking)

MAY:
- Check if the recipient account has RAM permissions
- Review invitation history for the resource share

## Phase 2 — Remediate

MUST:
- For organization sharing: enable RAM org sharing instead of using invitations
- For expired invitations: recreate the resource share or re-add the principal
- Verify recipient account ID is correct

SHOULD:
- Use organization sharing to avoid invitation management overhead
- Set up notifications for pending invitations
- Document the invitation acceptance process for recipients

MAY:
- Automate invitation monitoring
- Create runbooks for invitation acceptance

## Common Issues

- symptoms: "Invitation not received by recipient"
  diagnosis: "Organization sharing is enabled (no invitation needed) or account ID is wrong."
  resolution: "Check if org sharing is enabled. Verify account ID. Check recipient's RAM console."

- symptoms: "Invitation expired"
  diagnosis: "Invitation was not accepted within 7 days."
  resolution: "Remove and re-add the principal to send a new invitation."

## Output Format

```yaml
root_cause: "invitation_issue — <specific_cause>"
evidence:
  - type: invitation_status
    content: "<invitation status and details>"
  - type: org_sharing
    content: "<organization sharing status>"
severity: MEDIUM
mitigation:
  immediate: "Fix invitation issue or enable organization sharing"
  long_term: "Use organization sharing to eliminate invitation management"
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
