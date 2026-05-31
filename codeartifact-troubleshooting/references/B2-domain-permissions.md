---
title: "B2 — Domain Permission Issues"
description: "Diagnose CodeArtifact domain and repository permission policy problems"
status: active
severity: HIGH
triggers:
  - "domain permissions"
  - "access denied codeartifact"
  - "domain policy"
  - "repository policy"
  - "permission denied"
owner: devops-agent
objective: "Fix domain and repository permission policies for correct access control"
context: "CodeArtifact uses domain-level and repository-level resource policies alongside IAM policies. Cross-account access requires domain policies. Common issues include missing policies, wrong principals, and policy conflicts."
---

## Phase 1 — Triage

MUST:
- Check domain policy: `aws codeartifact get-domain-permissions-policy --domain <domain>`
- Check repository policy: `aws codeartifact get-repository-permissions-policy --domain <domain> --repository <repo>`
- Verify caller identity and IAM permissions: `aws sts get-caller-identity`
- Check CloudTrail for access denied events

SHOULD:
- Verify principal ARNs in policies match actual IAM entities
- Check for explicit deny statements
- Verify domain owner account

MAY:
- Use IAM Policy Simulator to test effective permissions
- Check for SCP restrictions

## Phase 2 — Remediate

MUST:
- Set domain policy for cross-account access:
  ```
  aws codeartifact put-domain-permissions-policy --domain <domain> --policy-document file://domain-policy.json
  ```
- Set repository policy for fine-grained access:
  ```
  aws codeartifact put-repository-permissions-policy --domain <domain> --repository <repo> --policy-document file://repo-policy.json
  ```

SHOULD:
- Use least-privilege principles in policies
- Separate read and write permissions

## Output Format

```yaml
root_cause: "domain_permissions — <specific_cause>"
evidence:
  - type: domain_policy
    content: "<domain permissions policy>"
  - type: repository_policy
    content: "<repository permissions policy>"
severity: HIGH
mitigation:
  immediate: "Fix domain or repository permissions policy"
  long_term: "Manage policies via IaC with policy validation"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Set domain policy for cross-account access | RED |
| Set repository policy for fine-grained access | RED |
| Use least-privilege principles | GREEN |
| Separate read and write permissions | GREEN |

## Escalation Conditions

- Fix requires modifying production domain or repository permission policies
- Fix involves granting cross-account access to package repositories

## Data Sensitivity

- **HIGH**: domain and repository policies control access to all packages and operations
- Cross-account policies expose trust relationships between AWS accounts

## Prohibited Actions

- NEVER suggest deleting upstream connections without understanding dependency impact
- NEVER suggest publishing over existing package versions

## Phase 3 — Rollback

1. If domain policy was modified, restore previous policy: `aws codeartifact put-domain-permissions-policy --domain <domain> --policy-document <previous-policy>`
2. If repository policy was modified, restore previous policy: `aws codeartifact put-repository-permissions-policy --domain <domain> --repository <repo> --policy-document <previous-policy>`
3. If policies were deleted, recreate from backup
4. Verify rollback by confirming access controls match expected permissions

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
