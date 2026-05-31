---
title: "D4 — Secrets Manager / SSM Parameter Store Issues"
description: "Diagnose task failures caused by inability to retrieve secrets or parameters"
status: active
severity: HIGH
triggers:
  - "ResourceInitializationError"
  - "unable to pull secrets"
  - "AccessDeniedException.*secretsmanager"
  - "AccessDeniedException.*ssm"
owner: devops-agent
objective: "Fix secrets/parameter retrieval for ECS tasks"
context: "ECS can inject secrets from Secrets Manager or SSM Parameter Store into containers as environment variables. The EXECUTION ROLE needs permissions to retrieve them. If secrets are KMS-encrypted, KMS decrypt permission is also needed."
---

## Phase 1 — Triage

MUST:
- Check task definition for secrets references in containerDefinitions[].secrets
- Verify execution role has secretsmanager:GetSecretValue or ssm:GetParameters
- Check if the secret/parameter exists and the ARN is correct
- If KMS-encrypted: verify execution role has kms:Decrypt for the KMS key

SHOULD:
- Check if the secret is in a different region or account
- Verify the secret value is valid (not empty or malformed)
- Check VPC endpoints if the task is in a private subnet (needs Secrets Manager/SSM endpoints)

MAY:
- Test secret retrieval manually with the execution role
- Check if the secret was recently rotated or deleted

## Common Issues

- symptoms: "ResourceInitializationError: unable to pull secrets or registry auth"
  diagnosis: "Execution role missing secretsmanager:GetSecretValue."
  resolution: "Add secretsmanager:GetSecretValue permission scoped to the specific secret ARN."

- symptoms: "Secret retrieval times out in private subnet"
  diagnosis: "No VPC endpoint for Secrets Manager and no NAT gateway."
  resolution: "Add VPC endpoint for com.amazonaws.region.secretsmanager."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
  - "Network configuration changes: YELLOW - May affect connectivity"
```

## Escalation Conditions

- Service handles production traffic or is business-critical
- Fix requires modifying the task definition for a running service
- Resolution involves changing capacity provider or cluster configuration
- Multiple services affected suggesting cluster-level or account-level issue
- Deployment rollback has been triggered by the circuit breaker

## Data Sensitivity

HIGH - ECS task definitions may contain secrets ARNs, environment variables with credentials, and sensitive configuration. Container logs may capture application data including PII. Service Connect and service discovery configurations may expose internal architecture.

## Prohibited Actions

- NEVER suggest stopping all tasks in a service to fix a deployment issue - this causes a full outage
- NEVER suggest removing or modifying the task execution role while tasks are running
- NEVER recommend deleting a service without first scaling desired count to 0 and draining connections
- NEVER suggest modifying a running task definition - create a new revision instead
- NEVER recommend disabling the deployment circuit breaker on a production service

## Phase 3 - Rollback

1. If task definition was updated, roll back to previous revision: `aws ecs update-service --cluster <cluster> --service <service> --task-definition <previous-revision>`
2. If service desired count was changed, restore original: `aws ecs update-service --cluster <cluster> --service <service> --desired-count <original>`
3. If IAM roles were modified, revert the task definition to the previous revision with correct role ARNs
4. If network configuration was changed, update service with original network settings: `aws ecs update-service --cluster <cluster> --service <service> --network-configuration <original>`
## Output Format

```yaml
root_cause: "secrets_ssm — <detail>"
evidence:
  - type: iam
    content: "<execution role permissions and secret ARN>"
severity: HIGH
mitigation:
  immediate: "Add required permissions or VPC endpoints"
  long_term: "Use VPC endpoints for secrets access, scope permissions to specific ARNs"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-tasks"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-services"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "execute-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest running containers as root to fix permission issues"
  - "NEVER suggest disabling task networking"
  - "NEVER remove IAM task role to troubleshoot"
