---
title: "A1 — Lambda Timeout"
description: "Diagnose Lambda function invocations that exceed the configured timeout"
status: active
severity: HIGH
triggers:
  - "Task timed out"
  - "Duration.*timeout"
  - "REPORT.*Timeout"
owner: devops-agent
objective: "Identify why the function is timing out and restore normal execution"
context: "Lambda timeout max is 900s. Timeouts occur when: downstream service is slow, code has infinite loops, VPC networking adds latency, cold start + execution exceeds limit, or timeout is set too low for the workload."
---

## Phase 1 — Triage

MUST:
- Check function timeout setting: `aws lambda get-function-configuration --function-name <name>` → Timeout
- Check CloudWatch Duration metric: is average duration approaching timeout?
- Check CloudWatch logs for "Task timed out after X seconds"
- Identify if timeout correlates with cold starts (check Init Duration in REPORT lines)

SHOULD:
- Check if function is in a VPC (VPC adds cold start latency and may block outbound calls)
- Check downstream service latency (API calls, DB queries, S3 operations)
- Check for retry storms (async invocations retrying timed-out calls)

MAY:
- Enable X-Ray tracing to identify the slow segment
- Check if function is waiting on a connection that never completes (DNS, TCP connect)

## Common Issues

- symptoms: "Task timed out, function is in VPC"
  diagnosis: "VPC Lambda cannot reach internet/AWS services without NAT gateway or VPC endpoints."
  resolution: "Add NAT gateway to VPC or use VPC endpoints for AWS services. Check security group outbound rules."

- symptoms: "Duration spikes on first invocation, then normal"
  diagnosis: "Cold start + initialization exceeds timeout. Common with large packages or VPC."
  resolution: "Increase timeout, reduce package size, use provisioned concurrency, or move initialization outside handler."

- symptoms: "Duration gradually increases over time"
  diagnosis: "Connection pool exhaustion, memory leak, or downstream service degradation."
  resolution: "Check connection reuse, review memory usage pattern, check downstream health."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
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

1. If function configuration was changed, revert using: `aws lambda update-function-configuration --function-name <name> --memory-size <original> --timeout <original>`
2. If reserved concurrency was modified, restore original value: `aws lambda put-function-concurrency --function-name <name> --reserved-concurrent-executions <original>`
3. If VPC configuration was changed, restore original VPC settings: `aws lambda update-function-configuration --function-name <name> --vpc-config SubnetIds=<original>,SecurityGroupIds=<original>`
4. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<downstream_slow|vpc_no_internet|cold_start|timeout_too_low|infinite_loop>"
evidence:
  - type: cloudwatch_logs
    content: "<timeout message and duration>"
severity: HIGH
mitigation:
  immediate: "Increase timeout or fix the blocking call"
  long_term: "Optimize code, use VPC endpoints, enable provisioned concurrency"
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
