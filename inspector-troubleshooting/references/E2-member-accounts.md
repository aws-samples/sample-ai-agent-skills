---
title: "E2 — Member Accounts"
description: "Diagnose member account issues in Inspector multi-account setup"
status: active
severity: MEDIUM
triggers:
  - "member account not scanning"
  - "account not associated"
  - "member enrollment failed"
  - "account status"
owner: devops-agent
objective: "Ensure member accounts are properly enrolled and scanning in Inspector"
context: "Member accounts must be associated with the delegated admin and have scanning enabled. Auto-enable can be configured for new accounts. Existing accounts may need manual enablement. Member account status can be ENABLED, DISABLED, or SUSPENDED."
---

## Phase 1 — Triage

MUST:
- List member accounts: `aws inspector2 list-members --query 'Members[*].{AccountId:AccountId,Status:RelationshipStatus}'`
- Check specific member: `aws inspector2 get-member --account-id <account-id>`
- Check member scanning status: `aws inspector2 batch-get-account-status --account-ids <account-id>`
- Check auto-enable configuration: `aws inspector2 describe-organization-configuration`

SHOULD:
- Verify the account is part of the organization
- Check if the member account has Inspector enabled locally
- Review member account IAM permissions

MAY:
- Check CloudTrail in the member account for Inspector events
- Verify service-linked role exists in member account

## Phase 2 — Remediate

MUST:
- Associate member account: `aws inspector2 associate-member --account-id <account-id>`
- Enable scanning for member: `aws inspector2 enable --account-ids <account-id> --resource-types EC2 ECR LAMBDA`
- Verify member status after enabling

SHOULD:
- Enable auto-enable for new accounts: `aws inspector2 update-organization-configuration --auto-enable '{"ec2":true,"ecr":true,"lambda":true}'`
- Batch enable for multiple accounts

MAY:
- Disassociate and re-associate if member is in error state
- Check member account service-linked role

## Common Issues

- symptoms: "Member account shows DISABLED"
  diagnosis: "Scanning not enabled for the member account."
  resolution: "Enable scanning from delegated admin account."

- symptoms: "New accounts not automatically enrolled"
  diagnosis: "Auto-enable not configured."
  resolution: "Update organization configuration with auto-enable for all resource types."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Associate member account | GREEN | Establishes management relationship |
| Enable scanning for member | YELLOW | Enables scanning in another account |
| Configure auto-enable | YELLOW | Automatically enables scanning in new accounts |

## Escalation Conditions

- Member account scanning not enabled for compliance — escalate to compliance team
- Batch enablement across many accounts — escalate to operations team
- Member account in error state — escalate to delegated admin

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws inspector2 list-members` | HIGH | Exposes member account IDs and status |
| `aws inspector2 get-member` | HIGH | Exposes member account details |
| `aws inspector2 batch-get-account-status` | MEDIUM | Exposes scanning status |

## Prohibited Actions

- **NEVER** suggest disabling Inspector scanning as a troubleshooting step
- **NEVER** suggest suppressing all findings without individual review
- **NEVER** disassociate member accounts without security team approval
- **NEVER** disable scanning for member accounts in production

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Associated member account | Disassociate: `aws inspector2 disassociate-member --account-id <id>` |
| Enabled scanning for member | Disable: `aws inspector2 disable --account-ids <id> --resource-types EC2 ECR LAMBDA` |
| Configured auto-enable | Disable auto-enable in organization configuration |

## Output Format

```yaml
root_cause: "member_account — <specific_cause>"
evidence:
  - type: member_status
    content: "<member account status>"
  - type: scanning_status
    content: "<scanning enabled/disabled per type>"
severity: MEDIUM
mitigation:
  immediate: "Enable scanning for the member account"
  long_term: "Configure auto-enable for new accounts"
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
  - "NEVER suggest disabling scanning to reduce findings"
  - "NEVER suggest ignoring critical vulnerabilities"
  - "NEVER suggest removing SSM agent to avoid scans"
