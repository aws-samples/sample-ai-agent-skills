---
title: "D2 — TLS/SigV4 Authentication"
description: "Diagnose Amazon Keyspaces TLS and SigV4 authentication issues"
status: active
severity: HIGH
triggers:
  - "authentication failed"
  - "SigV4 error"
  - "credential error"
  - "TLS certificate"
  - "access denied"
  - "service-specific credentials"
owner: devops-agent
objective: "Resolve TLS and authentication issues for Keyspaces connections"
context: "Keyspaces supports two authentication methods: IAM SigV4 (recommended) and service-specific credentials. SigV4 requires the SigV4 auth plugin for cqlsh or compatible driver. Service-specific credentials are generated via IAM console and tied to an IAM user. Both require TLS. IAM policies control Keyspaces access at the keyspace/table level."
---

## Phase 1 — Triage

MUST:
- Verify authentication method being used (SigV4 or service-specific credentials)
- Check IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <caller-arn> --action-names cassandra:Select cassandra:Modify`
- Verify TLS certificate is correctly configured
- Check if service-specific credentials are active: `aws iam list-service-specific-credentials --user-name <user> --service-name cassandra.amazonaws.com`

SHOULD:
- Verify SigV4 plugin is installed and configured correctly
- Check AWS credentials (access key, secret key, session token) are valid
- Verify the IAM policy grants access to the specific keyspace/table
- Check CloudTrail for authentication failure events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=ConsoleLogin --max-results 10`

MAY:
- Test with AWS CLI to verify IAM credentials work: `aws keyspaces list-keyspaces`
- Check for IAM policy conditions that may restrict access (IP, VPC, time)

## Phase 2 — Remediate

MUST:
- For SigV4: install and configure the SigV4 authentication plugin
- For service-specific credentials: generate new credentials if expired
- Ensure IAM policy includes required cassandra: actions for the target resources
- Verify TLS certificate chain is complete and trusted

SHOULD:
- Use IAM roles instead of long-term credentials where possible
- Rotate service-specific credentials regularly
- Use resource-level IAM policies for least-privilege access

MAY:
- Implement credential rotation automation
- Set up CloudTrail alerts for authentication failures

## Common Issues

- symptoms: "AuthenticationException with SigV4"
  diagnosis: "SigV4 plugin not installed, misconfigured, or AWS credentials invalid."
  resolution: "Install SigV4 auth plugin. Verify AWS credentials. Check region matches endpoint."

- symptoms: "Service-specific credentials rejected"
  diagnosis: "Credentials expired, deleted, or IAM user lacks Keyspaces permissions."
  resolution: "Generate new credentials. Verify IAM user has cassandra: permissions."

- symptoms: "Access denied on specific table but not others"
  diagnosis: "IAM policy restricts access to specific keyspace/table resources."
  resolution: "Update IAM policy to include the target keyspace/table ARN."

## Output Format

```yaml
root_cause: "authentication_failure — <specific_cause>"
evidence:
  - type: auth_method
    content: "<SigV4 or service-specific credentials>"
  - type: iam_policy
    content: "<IAM policy evaluation>"
  - type: tls_config
    content: "<TLS certificate status>"
severity: HIGH
mitigation:
  immediate: "Fix authentication configuration or IAM permissions"
  long_term: "Implement IAM roles and credential rotation"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never drop keyspaces or tables containing production data
- Never modify capacity mode on tables during peak traffic without load testing
- Never alter TTL settings without understanding data retention requirements
- Never change encryption settings on tables with active workloads
- Never disable point-in-time recovery on production tables

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore tables from point-in-time recovery if data is accidentally modified
- Revert capacity mode changes if throttling increases after modification
- Re-enable previous auto-scaling configurations if performance degrades

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
