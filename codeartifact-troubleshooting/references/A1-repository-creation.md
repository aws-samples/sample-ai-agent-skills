---
title: "A1 — Repository Creation Failures"
description: "Diagnose CodeArtifact repository creation failures including domain, naming, and permission issues"
status: active
severity: MEDIUM
triggers:
  - "create repository failed"
  - "repository creation error"
  - "codeartifact create-repository"
  - "repository already exists"
owner: devops-agent
objective: "Resolve repository creation failures and establish correct configuration"
context: "CodeArtifact repository creation requires a domain, proper IAM permissions, and valid naming. Common failures include domain not existing, naming conflicts, quota limits, and KMS key access issues."
---

## Phase 1 — Triage

MUST:
- Verify domain exists: `aws codeartifact describe-domain --domain <domain>`
- Check if repository already exists: `aws codeartifact describe-repository --domain <domain> --repository <repo>`
- Verify IAM permissions include `codeartifact:CreateRepository`
- Check service quotas: `aws service-quotas get-service-quota --service-code codeartifact --quota-code L-7E14AFBA`

SHOULD:
- Verify repository name follows naming rules (lowercase, alphanumeric, hyphens)
- Check KMS key access if domain uses custom KMS key
- Check CloudTrail for creation errors

MAY:
- Verify no SCP blocking CodeArtifact operations
- Check if domain owner account differs from current account

## Phase 2 — Remediate

MUST:
- If domain missing, create it first: `aws codeartifact create-domain --domain <domain>`
- If name conflict, use a different name
- If quota exceeded, request increase through Service Quotas

SHOULD:
- Configure upstream connections at creation time
- Set repository permissions policy after creation

## Output Format

```yaml
root_cause: "repository_creation_failure — <specific_cause>"
evidence:
  - type: domain_status
    content: "<domain details>"
  - type: error_message
    content: "<creation error>"
severity: MEDIUM
mitigation:
  immediate: "Fix domain, naming, or permissions"
  long_term: "Automate repository creation with IaC"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Create domain first if missing | YELLOW |
| Use a different name if conflict | GREEN |
| Request quota increase | GREEN |
| Configure upstream connections at creation | GREEN |
| Set repository permissions policy | YELLOW |

## Escalation Conditions

- Fix requires modifying production domain or repository configurations
- Fix involves changing repository permissions that affect package access

## Data Sensitivity

- **MEDIUM**: repository configurations expose package management topology and domain structure
- Repository permissions control access to internal packages and dependencies

## Prohibited Actions

- NEVER suggest deleting upstream connections without understanding dependency impact
- NEVER suggest publishing over existing package versions

## Phase 3 — Rollback

1. If repository was created incorrectly, delete it: `aws codeartifact delete-repository --domain <domain> --repository <repo>`
2. If domain was created, keep it (deleting a domain deletes all repositories)
3. If permissions policy was set incorrectly, update or delete it
4. Verify rollback by confirming `aws codeartifact describe-repository` shows expected configuration

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
