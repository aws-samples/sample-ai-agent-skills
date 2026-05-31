---
title: "E1 â€” Repository Policy Issues"
description: "Diagnose ECR repository policy misconfigurations including access control and policy syntax"
status: active
severity: HIGH
triggers:
  - "repository policy"
  - "ecr policy error"
  - "access denied ecr"
  - "policy syntax error"
  - "ecr permissions"
owner: devops-agent
objective: "Fix repository policy configuration to enable correct access control"
context: "ECR repository policies are resource-based policies that control access to specific repositories. They work alongside IAM policies. Common issues include syntax errors, wrong principal ARNs, missing actions, and policy conflicts with IAM."
---

## Phase 1 â€” Triage

MUST:
- Get current repository policy: `aws ecr get-repository-policy --repository-name <repo>`
- Verify principal ARNs are correct and exist
- Check that required actions are included (e.g., `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage`)
- Verify policy JSON syntax is valid

SHOULD:
- Compare with IAM policies on the calling principal
- Check for explicit deny statements that may override allows
- Verify condition keys are correct (e.g., `aws:PrincipalOrgID`)

MAY:
- Use IAM Policy Simulator to test effective permissions
- Check CloudTrail for access denied events with policy details

## Phase 2 â€” Remediate

MUST:
- Fix policy syntax errors and re-apply: `aws ecr set-repository-policy --repository-name <repo> --policy-text file://policy.json`
- Correct principal ARNs to match actual IAM entities
- Include all required ECR actions for the use case

SHOULD:
- Use least-privilege principles â€” grant only needed actions
- Use conditions to restrict access (e.g., org ID, source IP, VPC)

## Output Format

```yaml
root_cause: "repository_policy â€” <specific_cause>"
evidence:
  - type: policy_document
    content: "<repository policy>"
  - type: access_test
    content: "<access test result>"
severity: HIGH
mitigation:
  immediate: "Fix repository policy configuration"
  long_term: "Manage repository policies via IaC with policy validation"
```

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | HIGH — ECR hosts container images critical to deployments |
| Data Risk | MEDIUM — Images may contain application code and secrets |
| Recovery Complexity | MEDIUM — Deleted images can be rebuilt but cause deployment delays |
| Blast Radius | HIGH — Image issues affect all services pulling from the repository |

## Escalation Conditions

- Image pull failures affecting production deployments or auto-scaling events
- Repository policy changes locking out CI/CD pipelines from pushing images
- Cross-account replication failures causing stale images in secondary regions
- Image scanning revealing critical CVEs in production-running images
- Lifecycle policy unexpectedly deleting images referenced by running tasks
- Encryption key issues preventing image push or pull operations

## Data Sensitivity

- Container images may embed application secrets, API keys, or credentials
- Repository policies may reveal cross-account trust relationships
- Image tags and digests are deployment identifiers — treat as operational metadata
- Scan results may expose vulnerability details that should not be shared broadly
- ECR auth tokens grant broad repository access — treat as short-lived secrets

## PROHIBITED Actions

- NEVER suggest deleting images that are referenced by running tasks or services
- NEVER modify repository policies without verifying CI/CD pipeline access requirements
- NEVER disable tag immutability on repositories enforcing deployment traceability
- NEVER delete a repository without confirming all dependent services have been migrated
- NEVER share or log ECR authentication tokens in plain text
- NEVER disable image scanning on repositories subject to compliance requirements

## Phase 3 — Rollback

- If image was accidentally deleted: rebuild and push from CI/CD pipeline or pull from replication target
- If repository policy change blocks access: revert to previous policy JSON from version control
- If lifecycle policy deleted needed images: restore from cross-region replica or rebuild from source
- If tag immutability was disabled: re-enable and audit for any overwritten tags
- If encryption configuration was changed: verify KMS key access and re-push affected images
- Document all rollback actions with image digests and tags for audit trail
## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN â€” read-only"
  - "Phase 2 configuration changes: YELLOW â€” state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED â€” destructive or irreversible"

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
