---
title: "E2 — Cross-Account Access"
description: "Diagnose CodeArtifact cross-account access issues including domain policies and IAM configuration"
status: active
severity: HIGH
triggers:
  - "cross-account codeartifact"
  - "access denied other account"
  - "share domain"
  - "cross-account repository"
owner: devops-agent
objective: "Enable secure cross-account CodeArtifact access"
context: "Cross-account CodeArtifact access requires a domain permissions policy granting access to the remote account AND IAM permissions in the remote account. The --domain-owner parameter must specify the domain owner account ID."
---

## Phase 1 — Triage

MUST:
- Check domain policy: `aws codeartifact get-domain-permissions-policy --domain <domain>`
- Verify remote account is listed in domain policy
- In remote account, verify IAM permissions include CodeArtifact actions
- Verify `--domain-owner` is used in CLI commands from remote account

SHOULD:
- Check if KMS key policy grants access to remote account
- Verify repository permissions policy if applicable
- Check CloudTrail in both accounts for denied events

MAY:
- Test token generation from remote account: `aws codeartifact get-authorization-token --domain <domain> --domain-owner <owner-account>`
- Verify no SCP blocking cross-account access

## Phase 2 — Remediate

MUST:
- Set domain policy granting cross-account access:
  ```
  aws codeartifact put-domain-permissions-policy --domain <domain> --policy-document '{
    "Version":"2012-10-17",
    "Statement":[{
      "Effect":"Allow",
      "Principal":{"AWS":"arn:aws:iam::<remote-account>:root"},
      "Action":["codeartifact:GetAuthorizationToken","codeartifact:GetRepositoryEndpoint","codeartifact:ReadFromRepository"],
      "Resource":"*"
    }]
  }'
  ```
- In remote account, add IAM permissions for CodeArtifact and `sts:GetServiceBearerToken`
- If KMS encrypted, add remote account to KMS key policy

SHOULD:
- Use specific principals instead of account root
- Use Organizations condition for org-wide access

## Output Format

```yaml
root_cause: "cross_account_access — <specific_cause>"
evidence:
  - type: domain_policy
    content: "<domain permissions policy>"
  - type: iam_policy
    content: "<remote account IAM policy>"
severity: HIGH
mitigation:
  immediate: "Configure domain policy and IAM permissions"
  long_term: "Use Organizations-based policies for multi-account access"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Set domain policy granting cross-account access | RED |
| Add IAM permissions in remote account | YELLOW |
| Add remote account to KMS key policy | RED |
| Use specific principals instead of account root | GREEN |
| Use Organizations condition for org-wide access | YELLOW |

## Escalation Conditions

- Fix requires modifying production domain policies for cross-account access
- Fix involves granting KMS key access to external accounts

## Data Sensitivity

- **HIGH**: cross-account configurations expose trust relationships between AWS accounts
- Domain policies and KMS key policies control package access across organizational boundaries

## Prohibited Actions

- NEVER suggest deleting upstream connections without understanding dependency impact
- NEVER suggest publishing over existing package versions

## Phase 3 — Rollback

1. If domain policy was modified, restore previous policy: `aws codeartifact put-domain-permissions-policy --domain <domain> --policy-document <previous-policy>`
2. If KMS key policy was changed, restore previous key policy
3. If IAM permissions were added in remote account, revert to previous policy
4. Verify rollback by confirming cross-account access matches expected permissions

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
