---
title: "F2 — App Runner Secrets and Environment Variables Issues"
description: "Diagnose environment variable and secrets configuration problems"
status: active
severity: MEDIUM
triggers:
  - "environment variable"
  - "env var"
  - "secret"
  - "SSM parameter"
  - "Secrets Manager"
  - "missing variable"
  - "configuration"
owner: devops-agent
objective: "Identify and fix environment variable and secrets configuration issues"
context: "App Runner supports plain environment variables and secrets (referencing SSM Parameter Store or Secrets Manager). Environment variables are visible in the service configuration. Secrets are resolved at runtime. The instance role must have permission to read secrets. Build-time and runtime environment variables are separate configurations for code-based sources."
---

## Phase 1 — Triage

MUST:
- Check runtime environment variables and secrets: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.SourceConfiguration.ImageRepository.ImageConfiguration'` (for ECR) or check CodeConfiguration for code sources
- Review application logs for missing variable errors: `aws logs get-log-events --log-group-name /aws/apprunner/<service-name>/<service-id>/application --log-stream-name <stream>`
- If using secrets, check instance role permissions for SSM/Secrets Manager
- Verify the SSM parameter or Secrets Manager secret exists: `aws ssm get-parameter --name <param-name>` or `aws secretsmanager describe-secret --secret-id <secret-id>`
- Check if the secret ARN format is correct in the service configuration

SHOULD:
- Distinguish between build environment variables and runtime environment variables (code sources only)
- Verify the instance role has `ssm:GetParameters` for SSM or `secretsmanager:GetSecretValue` for Secrets Manager
- Check if the secret is in the same region as the App Runner service

MAY:
- Check if KMS permissions are needed to decrypt the secret
- Verify no resource policy on the secret is blocking access
- Check CloudTrail for secret access denied events

## Phase 2 — Remediate

MUST:
- Add missing environment variables via service update
- For secrets, ensure the instance role has read permissions for the referenced SSM parameter or Secrets Manager secret
- Verify the secret ARN or SSM parameter name is correct in the configuration

SHOULD:
- Use secrets for sensitive values instead of plain environment variables
- Configure build environment variables separately from runtime variables for code sources
- Use specific SSM parameter versions or Secrets Manager version stages for consistency

MAY:
- Implement secret rotation with Secrets Manager
- Use parameter hierarchies in SSM for organized configuration management
- Add KMS key permissions if using customer-managed KMS keys for secret encryption

## Common Issues

- symptoms: "Application crashes with 'undefined' or 'missing env var' error"
  diagnosis: "Environment variable not configured in the App Runner service."
  resolution: "Add the environment variable via update-service API or console."

- symptoms: "Secret value is the ARN string instead of the actual secret value"
  diagnosis: "Secret is configured as a plain environment variable instead of a secret reference."
  resolution: "Configure it as a secret (not an environment variable) with the SSM/Secrets Manager ARN."

- symptoms: "Secret resolution fails with access denied"
  diagnosis: "Instance role lacks ssm:GetParameters or secretsmanager:GetSecretValue permission."
  resolution: "Add the required permission to the instance role. Include kms:Decrypt if using CMK."

## Output Format

```yaml
root_cause: "secrets_env_vars — <specific_cause>"
evidence:
  - type: env_config
    content: "<environment variables and secrets configuration>"
  - type: instance_role
    content: "<role permissions for secret access>"
  - type: secret_status
    content: "<SSM parameter or Secrets Manager secret status>"
severity: MEDIUM
mitigation:
  immediate: "Fix environment variable or secret configuration"
  long_term: "Use secrets for sensitive values and implement rotation"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Add missing environment variables | YELLOW | Service configuration change |
| Ensure instance role has secret read permissions | YELLOW | IAM policy change |
| Verify secret ARN/parameter name | GREEN | Read-only check |
| Use secrets for sensitive values | GREEN | Security best practice |
| Implement secret rotation | GREEN | Security improvement |

## Escalation Conditions

- Fix requires redeployment of production service
- NEVER suggest scaling to 0 (not supported)

## Data Sensitivity

- **Environment variables**: Application configuration — INTERNAL
- **Secrets**: SSM parameters and Secrets Manager values — CONFIDENTIAL
- **Instance role**: IAM permissions for secret access — INTERNAL

## Prohibited Actions

- NEVER suggest hardcoding credentials in environment variables
- NEVER suggest scaling to 0 instances (not supported by App Runner)
- NEVER log secret values in application output

## Phase 3 — Rollback

1. Revert environment variable changes via update-service
2. Revert IAM policy changes on the instance role
3. Restore previous secret values if they were modified
4. Verify application starts correctly with reverted configuration
5. Monitor application logs for missing variable errors

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
