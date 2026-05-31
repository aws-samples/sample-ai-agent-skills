---
title: "E1 — Domain Configuration Issues"
description: "Diagnose CodeArtifact domain configuration problems including encryption, naming, and limits"
status: active
severity: MEDIUM
triggers:
  - "domain configuration"
  - "create domain failed"
  - "domain encryption"
  - "domain error"
  - "domain limit"
owner: devops-agent
objective: "Resolve domain configuration issues and establish correct domain setup"
context: "CodeArtifact domains are the top-level organizational unit. All repositories in a domain share the same KMS key. Common issues include domain creation failures, KMS key access, and quota limits."
---

## Phase 1 — Triage

MUST:
- Check domain status: `aws codeartifact describe-domain --domain <domain>`
- Verify KMS key if custom encryption: `aws kms describe-key --key-id <key-id>`
- Check domain quota: `aws service-quotas get-service-quota --service-code codeartifact --quota-code L-4B4E2F57`
- List existing domains: `aws codeartifact list-domains`

SHOULD:
- Verify domain naming follows rules
- Check KMS key policy grants CodeArtifact access
- Verify IAM permissions for domain operations

MAY:
- Check CloudTrail for domain-related errors
- Verify no SCP blocking domain creation

## Phase 2 — Remediate

MUST:
- If KMS error, grant CodeArtifact service access to the KMS key
- If quota exceeded, request increase
- If domain creation failed, check error message and fix root cause

SHOULD:
- Plan domain structure around encryption and billing boundaries
- Use separate domains for different security requirements

## Output Format

```yaml
root_cause: "domain_configuration — <specific_cause>"
evidence:
  - type: domain_status
    content: "<domain details>"
  - type: kms_status
    content: "<KMS key details>"
severity: MEDIUM
mitigation:
  immediate: "Fix domain configuration or KMS access"
  long_term: "Document domain architecture and manage via IaC"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Grant CodeArtifact service access to KMS key | YELLOW |
| Request quota increase | GREEN |
| Fix domain creation error | GREEN |
| Plan domain structure around encryption boundaries | GREEN |
| Use separate domains for different security requirements | GREEN |

## Escalation Conditions

- Fix requires modifying production KMS key policies
- Fix involves domain-level changes that affect all repositories

## Data Sensitivity

- **HIGH**: domain configurations include KMS encryption settings for all repositories
- KMS key policies control who can access encrypted package data

## Prohibited Actions

- NEVER suggest deleting upstream connections without understanding dependency impact
- NEVER suggest publishing over existing package versions

## Phase 3 — Rollback

1. If KMS key policy was modified, restore previous key policy
2. Domain deletion is destructive (deletes all repositories) — avoid unless absolutely necessary
3. If quota increase was requested, no rollback needed
4. Verify rollback by confirming `aws codeartifact describe-domain` shows expected configuration

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
