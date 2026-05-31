---
title: "A2 — Lambda Out of Memory"
description: "Diagnose Lambda OOM errors when function exceeds configured memory"
status: active
severity: HIGH
triggers:
  - "Runtime exited with error.*memory"
  - "REPORT.*Max Memory Used.*Memory Size"
  - "Cannot allocate memory"
  - "signal: killed"
owner: devops-agent
objective: "Right-size memory allocation to prevent OOM kills"
context: "Lambda memory setting controls BOTH memory AND proportional CPU. 1769 MB = 1 full vCPU. OOM kills happen when the function uses more memory than allocated. Memory also affects CPU performance — low memory = slow CPU."
---

## Phase 1 — Triage

MUST:
- Check function memory setting: `aws lambda get-function-configuration` → MemorySize
- Check REPORT lines in CloudWatch logs: compare "Max Memory Used" vs "Memory Size"
- If Max Memory Used ≥ 90% of Memory Size: function is at risk of OOM
- Check for "Runtime exited with error" or "signal: killed" in logs

SHOULD:
- Check if memory usage grows over warm invocations (memory leak)
- Check if specific input sizes trigger OOM (large payloads, file processing)

MAY:
- Use Lambda Power Tuning to find optimal memory/cost balance
- Check if the function processes large files in memory vs streaming

## Common Issues

- symptoms: "REPORT shows Max Memory Used equals Memory Size"
  diagnosis: "Function hit the memory limit. Lambda killed the process."
  resolution: "Increase memory. Remember: more memory = more CPU = potentially faster execution."

- symptoms: "Memory usage grows with each warm invocation"
  diagnosis: "Memory leak in global scope. Objects accumulating between invocations."
  resolution: "Fix the leak (clear caches, close connections). Or reduce maxAge to force cold starts."

- symptoms: "Function is slow AND using near-max memory"
  diagnosis: "Low memory = low CPU. Function may be CPU-bound but starved."
  resolution: "Increase memory to get more CPU. 1769 MB = 1 vCPU. Test with Lambda Power Tuning."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Cache invalidation: YELLOW - Temporarily increases origin load"
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
2. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<memory_too_low|memory_leak|large_payload|cpu_starved>"
evidence:
  - type: report_line
    content: "<REPORT showing Max Memory Used vs Memory Size>"
severity: HIGH
mitigation:
  immediate: "Increase memory allocation"
  long_term: "Use Lambda Power Tuning, fix memory leaks, stream large files"
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
