---
title: "B1 — Cold Start Latency"
description: "Diagnose and reduce Lambda cold start duration"
status: active
severity: MEDIUM
triggers:
  - "Init Duration"
  - "cold start"
  - "first invocation.*slow"
owner: devops-agent
objective: "Identify cold start contributors and reduce initialization time"
context: "Cold starts occur when Lambda creates a new execution environment. Init Duration in REPORT lines shows cold start time. Contributors: runtime init, package size, global scope code, VPC ENI attachment, and layer extraction."
---

## Phase 1 — Triage

MUST:
- Check REPORT lines for "Init Duration" — this is the cold start time
- Check deployment package size: `aws lambda get-function --function-name <name>` → CodeSize
- Check number and size of layers
- Check if function is in a VPC

SHOULD:
- Compare Init Duration vs total Duration to understand cold start impact
- Check runtime (Java/.NET have longer cold starts than Python/Node.js)
- Check global scope initialization (SDK clients, DB connections, config loading)

## Common Issues

- symptoms: "Init Duration > 5 seconds, Java runtime"
  diagnosis: "JVM startup + class loading is inherently slow. Normal for Java."
  resolution: "Use SnapStart (Java 11+), reduce classpath, use GraalVM native image, or switch to lighter runtime."

- symptoms: "Init Duration high, large deployment package"
  diagnosis: "Large package = more time to extract and load."
  resolution: "Reduce package size. Remove unused dependencies. Use layers for shared code."

- symptoms: "Init Duration includes VPC setup time"
  diagnosis: "VPC Lambda uses Hyperplane ENIs. First cold start in a new ENI combo takes longer."
  resolution: "Use provisioned concurrency. VPC cold starts are much better than pre-2019 but still add time."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
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
3. If layers were changed, revert to previous layer versions: `aws lambda update-function-configuration --function-name <name> --layers <previous-layer-arns>`
4. If VPC configuration was changed, restore original VPC settings: `aws lambda update-function-configuration --function-name <name> --vpc-config SubnetIds=<original>,SecurityGroupIds=<original>`
5. If environment variables were changed, restore previous values: `aws lambda update-function-configuration --function-name <name> --environment Variables=<original>`
6. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<large_package|vpc_eni|runtime_init|global_scope|layers>"
evidence:
  - type: report_line
    content: "<REPORT showing Init Duration>"
severity: MEDIUM
mitigation:
  immediate: "Use provisioned concurrency for latency-sensitive functions"
  long_term: "Reduce package size, optimize init code, consider SnapStart (Java)"
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
