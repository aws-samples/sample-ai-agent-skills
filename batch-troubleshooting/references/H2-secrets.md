---
title: "H2 — Secrets Access Issues"
description: "Diagnose secrets and sensitive data access problems in Batch jobs"
status: active
severity: MEDIUM
triggers:
  - "secret not found"
  - "secrets manager error"
  - "parameter store error"
  - "sensitive data"
  - "credential access"
owner: devops-agent
objective: "Identify and resolve secrets access issues"
context: "Secrets access issues include: execution role missing Secrets Manager/SSM permissions, secret ARN wrong, secret not in same region, or KMS key permissions for encrypted secrets."
---

## Phase 1 — Triage

MUST:
- Check job definition secrets configuration: `aws batch describe-job-definitions --job-definitions <def-name> --query 'jobDefinitions[*].containerProperties.secrets'`
- Verify execution role has secrets permissions
- Check secret exists: `aws secretsmanager describe-secret --secret-id <secret-name>` or `aws ssm get-parameter --name <param-name>`
- Verify secret ARN format is correct

SHOULD:
- Check KMS key permissions if secret is encrypted with CMK
- Verify secret is in the same region as the Batch job
- Check for secret version/stage issues

MAY:
- Check for secret rotation issues
- Review secret access in CloudTrail

## Phase 2 — Remediate

MUST:
- Add secrets permissions to execution role: secretsmanager:GetSecretValue or ssm:GetParameters
- Fix secret ARN in job definition
- Add KMS decrypt permission if using CMK

SHOULD:
- Configure secrets in job definition: `"secrets":[{"name":"DB_PASSWORD","valueFrom":"arn:aws:secretsmanager:<region>:<account>:secret:<name>"}]`
- Use Secrets Manager for credentials, SSM Parameter Store for configuration

MAY:
- Implement secret rotation
- Use fine-grained secret access policies

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Add secrets permissions to execution role | YELLOW | Permission change — review scope |
| Fix secret ARN in job definition | GREEN | Configuration fix — non-destructive |
| Add KMS decrypt permission for CMK | YELLOW | Expands key access — review scope |
| Configure secrets in job definition | GREEN | Configuration — non-destructive |
| Implement secret rotation | GREEN | Security improvement — non-destructive |

## Escalation Conditions

- Compute environment runs production batch jobs
- Secrets access failure blocks all job executions
- Secret rotation causes temporary access disruption
- KMS key permissions affect other services using the same key

## Data Sensitivity

- **Classification: HIGH**
- Secrets configuration reveals credential management architecture
- Secret ARNs expose which secrets are used by which jobs
- KMS key permissions reveal encryption trust boundaries
- Execution role permissions expose secrets access scope

## Prohibited Actions

- **NEVER** suggest deleting a compute environment with running jobs
- **NEVER** suggest hardcoding secrets in environment variables instead of using Secrets Manager
- **NEVER** suggest granting broad secrets access (secretsmanager:*) to execution roles
- **NEVER** suggest disabling secret encryption to resolve access issues

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Added secrets permissions to execution role | Remove added permissions if not needed |
| Updated secret ARN in job definition | Revert to previous job definition revision |
| Added KMS decrypt permission | Remove KMS permission if not needed |

## Output Format

```yaml
root_cause: "secrets_access_issue — <specific_cause>"
evidence:
  - type: secrets_config
    content: "<secrets configuration>"
  - type: permissions
    content: "<execution role permissions>"
severity: MEDIUM
mitigation:
  immediate: "Fix execution role permissions or secret ARN"
  long_term: "Implement proper secrets management with rotation"
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
  - "NEVER suggest overly broad compute environment permissions"
  - "NEVER suggest disabling job queue scheduling"
  - "NEVER suggest public subnet placement for compute environments"
