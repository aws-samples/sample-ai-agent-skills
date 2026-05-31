---
title: "Z1 — General Lambda Troubleshooting (Catch-All)"
description: "Fallback SOP for Lambda issues that don't match a specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown Lambda issue and classify the failure domain"
context: "This SOP is invoked when symptoms don't match specific runbooks. It provides a broad investigation that narrows the failure domain."
---

## Phase 1 — Triage

MUST:
- Get function configuration: `aws lambda get-function-configuration --function-name <name>`
- Check CloudWatch metrics: Invocations, Errors, Throttles, Duration, ConcurrentExecutions
- Check recent CloudWatch logs: `aws logs filter-log-events --log-group-name /aws/lambda/<name> --filter-pattern "ERROR"`
- Check event source mappings: `aws lambda list-event-source-mappings --function-name <name>`

## Phase 2 — Classify

Based on triage, classify into:
- Invocation errors → A1-A4
- Cold start issues → B1-B3
- Throttling → C1-C3
- VPC networking → D1-D4
- Permissions → E1-E4
- Deployment → F1-F3
- Event sources → G1-G4
- Performance → H1-H3
- Observability → I1-I2

If classified: switch to specific SOP. If unclassified: escalate with all evidence.


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Adjust scaling/concurrency: YELLOW - May impact availability if misconfigured"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
  - "Network configuration changes: YELLOW - May affect connectivity"
```

## Escalation Conditions

- Function serves a production API or critical workload
- Fix requires changing reserved concurrency settings
- Function is processing sensitive data (PII, financial, healthcare)
- Resolution involves modifying VPC configuration or security groups
- Multiple functions affected suggesting account-level issue

## Data Sensitivity

HIGH - Lambda function code and environment variables may contain secrets, API keys, database credentials, and encryption keys. CloudWatch logs may capture sensitive request/response data. X-Ray traces may contain PII in segment metadata.

## Prohibited Actions

- NEVER suggest setting reserved concurrency to 0 - this effectively disables the function
- NEVER suggest deleting a function alias that is serving live traffic
- NEVER recommend removing or replacing the execution role on a running function without verifying the new role has equivalent permissions
- NEVER suggest publishing function code changes directly to a production alias without testing
- NEVER expose environment variable values in logs or diagnostic output - they may contain secrets

## Phase 3 - Rollback

1. If function configuration was changed, revert using: `aws lambda update-function-configuration --function-name <name> --memory-size <original> --timeout <original>`
2. If reserved concurrency was modified, restore original value: `aws lambda put-function-concurrency --function-name <name> --reserved-concurrent-executions <original>`
3. If a new version was published, point the alias back to the previous version: `aws lambda update-alias --function-name <name> --name <alias> --function-version <previous>`
4. If VPC configuration was changed, restore original VPC settings: `aws lambda update-function-configuration --function-name <name> --vpc-config SubnetIds=<original>,SecurityGroupIds=<original>`
5. If execution role was modified, revert IAM policy changes and verify function can still access required resources
6. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<invocation|cold_start|concurrency|networking|permissions|deployment|event_source|performance|observability|unknown>"
investigation_path: "get-function-configuration → metrics → logs → classification"
severity: MEDIUM
mitigation:
  immediate: "<specific action or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
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
  - command: "get-function-configuration"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-policy"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "invoke"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest Resource: * in Lambda execution role"
  - "NEVER suggest disabling VPC configuration to fix connectivity"
  - "NEVER expose function URLs without authentication"
