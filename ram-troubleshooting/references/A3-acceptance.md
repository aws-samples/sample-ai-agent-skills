---
title: "A3 — Invitation Acceptance Issues"
description: "Diagnose RAM invitation acceptance failures"
status: active
severity: MEDIUM
triggers:
  - "cannot accept invitation"
  - "acceptance failed"
  - "AcceptResourceShareInvitation error"
  - "invitation rejected"
owner: devops-agent
objective: "Identify and resolve RAM invitation acceptance issues"
context: "Invitation acceptance can fail due to IAM permissions, expired invitations, already accepted/rejected invitations, or SCP restrictions. The recipient needs ram:AcceptResourceShareInvitation permission. Organization sharing bypasses invitations entirely."
---

## Phase 1 — Triage

MUST:
- Check invitation status: `aws ram get-resource-share-invitations --query 'resourceShareInvitations[*].{Arn:resourceShareInvitationArn,Status:status,Sender:senderAccountId}'`
- Verify IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <caller-arn> --action-names ram:AcceptResourceShareInvitation`
- Check if invitation is still PENDING (not expired or already processed)
- Verify the caller is in the recipient account

SHOULD:
- Check for SCPs blocking RAM acceptance
- Verify the invitation ARN is correct
- Check CloudTrail for acceptance attempts
- List pending invitation resources: `aws ram list-pending-invitation-resources --resource-share-invitation-arn <arn>`

MAY:
- Check if the resource share still exists on the sender side
- Verify network connectivity to RAM API endpoint

## Phase 2 — Remediate

MUST:
- Fix IAM permissions to include ram:AcceptResourceShareInvitation
- Accept the invitation: `aws ram accept-resource-share-invitation --resource-share-invitation-arn <arn>`
- For expired invitations, ask sender to re-share

SHOULD:
- Verify shared resources are accessible after acceptance
- Document the acceptance process
- Consider enabling organization sharing to avoid invitations

MAY:
- Automate invitation acceptance for trusted accounts
- Set up alerts for pending invitations

## Common Issues

- symptoms: "AccessDeniedException when accepting"
  diagnosis: "Caller lacks ram:AcceptResourceShareInvitation permission."
  resolution: "Add ram:AcceptResourceShareInvitation to the caller's IAM policy."

- symptoms: "Invitation already processed"
  diagnosis: "Invitation was already accepted or rejected."
  resolution: "Check invitation status. If rejected, ask sender to re-share."

## Output Format

```yaml
root_cause: "acceptance_failure — <specific_cause>"
evidence:
  - type: invitation_status
    content: "<invitation status>"
  - type: iam_permissions
    content: "<IAM policy evaluation>"
severity: MEDIUM
mitigation:
  immediate: "Fix permissions and accept invitation"
  long_term: "Enable organization sharing to eliminate invitation workflow"
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
