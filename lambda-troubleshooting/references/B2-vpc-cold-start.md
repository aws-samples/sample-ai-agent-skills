---
title: "B2 — VPC Cold Start"
description: "Diagnose additional cold start latency for VPC-attached Lambda functions"
status: active
severity: MEDIUM
triggers:
  - "VPC.*cold start"
  - "ENI.*creation"
  - "Init Duration.*VPC"
owner: devops-agent
objective: "Minimize VPC-related cold start overhead"
context: "Since 2019, VPC Lambda uses Hyperplane ENIs (shared, pre-created). Cold starts improved from 10-30s to ~1s additional. But first invocation in a new subnet/SG combination still requires ENI setup. Provisioned concurrency eliminates this entirely."
---

## Common Issues

- symptoms: "First invocation after deployment takes 5-10 seconds, subsequent are fast"
  diagnosis: "Hyperplane ENI being set up for the first time in this subnet/SG combo."
  resolution: "Normal one-time cost. Use provisioned concurrency for consistent latency."

- symptoms: "All invocations are slow, function is in VPC"
  diagnosis: "Not a cold start issue. Likely VPC networking problem (no NAT, SG blocking, DNS)."
  resolution: "Check VPC connectivity. See D1 (VPC no internet) runbook."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Adjust scaling/concurrency: YELLOW - May impact availability if misconfigured"
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

1. If reserved concurrency was modified, restore original value: `aws lambda put-function-concurrency --function-name <name> --reserved-concurrent-executions <original>`
2. If a new version was published, point the alias back to the previous version: `aws lambda update-alias --function-name <name> --name <alias> --function-version <previous>`
3. If VPC configuration was changed, restore original VPC settings: `aws lambda update-function-configuration --function-name <name> --vpc-config SubnetIds=<original>,SecurityGroupIds=<original>`
4. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<hyperplane_eni_setup|vpc_networking|first_subnet_sg_combo>"
severity: MEDIUM
mitigation:
  immediate: "Use provisioned concurrency"
  long_term: "Minimize unique subnet/SG combinations, consider removing VPC if not needed"
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
