---
title: "G2 — MWAA Secrets Backend Issues"
description: "Diagnose secrets backend configuration problems in MWAA"
status: active
severity: MEDIUM
triggers:
  - "secrets backend"
  - "connection not found"
  - "variable not found"
  - "Secrets Manager"
  - "Parameter Store"
owner: devops-agent
objective: "Identify and resolve secrets backend configuration issues in MWAA"
context: "MWAA can use AWS Secrets Manager or Systems Manager Parameter Store as a secrets backend for Airflow connections and variables. Incorrect naming conventions, missing IAM permissions, or misconfigured backend settings cause silent lookup failures."
---

## Phase 1 — Triage

MUST:
- Check secrets backend configuration: `aws mwaa get-environment --name <env-name> --query 'Environment.AirflowConfigurationOptions'`
- Look for secrets_backend settings in configuration overrides
- Verify secrets exist with correct naming: `aws secretsmanager list-secrets --filters Key=name,Values=airflow/connections/`
- Check execution role has Secrets Manager permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names secretsmanager:GetSecretValue --resource-arns arn:aws:secretsmanager:<region>:<account>:secret:airflow/*`
- Check task logs for connection lookup errors: `aws logs filter-log-events --log-group-name airflow-<env-name>-Task --filter-pattern "connection" --start-time <epoch-ms>`

SHOULD:
- Verify the prefix pattern matches the secret names (default: `airflow/connections/` and `airflow/variables/`)
- Check if secrets are encrypted with a CMK and execution role has KMS decrypt permission
- Verify secret values are in the correct format (URI or JSON for connections)

MAY:
- Test secret retrieval directly: `aws secretsmanager get-secret-value --secret-id airflow/connections/<conn_id>`
- Check if Parameter Store is used instead of Secrets Manager

## Phase 2 — Remediate

MUST:
- Configure secrets backend in Airflow configuration overrides:
  - `secrets.backend`: `airflow.providers.amazon.aws.secrets.secrets_manager.SecretsManagerBackend`
  - `secrets.backend_kwargs`: `{"connections_prefix": "airflow/connections", "variables_prefix": "airflow/variables"}`
- Grant `secretsmanager:GetSecretValue` to execution role
- Use correct naming convention: `airflow/connections/<connection_id>`

SHOULD:
- Store connection values in JSON format for clarity
- Grant KMS decrypt permission if using CMK encryption
- Test secret retrieval before relying on it in DAGs

MAY:
- Use separate prefixes for different environments (dev/staging/prod)
- Implement secret rotation with Secrets Manager rotation lambdas

## Common Issues

- symptoms: "Connection not found despite secret existing in Secrets Manager"
  diagnosis: "Secret name doesn't match the configured prefix pattern."
  resolution: "Verify prefix in backend_kwargs matches the secret naming convention."

- symptoms: "Secrets backend configured but Airflow uses metastore connections"
  diagnosis: "Secrets backend configuration override not applied correctly."
  resolution: "Verify AIRFLOW__SECRETS__BACKEND and AIRFLOW__SECRETS__BACKEND_KWARGS are set."

- symptoms: "AccessDeniedException when reading secrets"
  diagnosis: "Execution role lacks secretsmanager:GetSecretValue permission."
  resolution: "Add Secrets Manager read permissions to the execution role."

## Output Format

```yaml
root_cause: "secrets_backend_issue — <specific_cause>"
evidence:
  - type: backend_config
    content: "<secrets backend configuration>"
  - type: secret_names
    content: "<secret naming and prefix pattern>"
  - type: iam_permissions
    content: "<execution role Secrets Manager permissions>"
severity: MEDIUM
mitigation:
  immediate: "Fix naming convention or IAM permissions"
  long_term: "Standardize secrets management across all MWAA environments"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — secrets backend stores sensitive credentials and connection strings
- Service Disruption Risk: MEDIUM — secrets backend failures prevent tasks from accessing credentials
- Compliance Impact: HIGH — secrets management directly affects security compliance

## Escalation Conditions
- Secrets backend becomes completely inaccessible
- Secrets exposed in logs or task output
- Unauthorized access to secrets detected
- Secrets Manager or SSM Parameter Store permissions revoked
- Secrets rotation causes widespread task failures

## Data Sensitivity
- Connection strings and credentials — RESTRICTED
- API keys and tokens — RESTRICTED
- Database passwords — RESTRICTED
- Secrets Manager ARNs — CONFIDENTIAL
- SSM Parameter Store paths — CONFIDENTIAL
- Never expose secret values in logs, outputs, or error messages

## Prohibited Actions
- Do NOT store secrets in Airflow Variables instead of the secrets backend
- Do NOT log secret values in DAG code or task output
- Do NOT grant broad Secrets Manager or SSM access to the execution role
- Do NOT disable secrets backend encryption
- Do NOT share secrets across environments with different security classifications
- Do NOT bypass secrets backend by hardcoding credentials in DAGs

## Phase 3 — Rollback
- Restore secrets backend configuration to previous working state
- Rotate any secrets that may have been exposed during the incident
- Restore execution role permissions for Secrets Manager or SSM access
- Verify all tasks can retrieve secrets after configuration changes
- Audit secrets access logs for unauthorized access attempts
- Document the incident and update secrets management procedures

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
  - "NEVER suggest disabling webserver authentication"
  - "NEVER suggest public network access mode for production"
  - "NEVER suggest overly broad Airflow execution role"
