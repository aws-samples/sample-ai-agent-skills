---
title: "A4 — Runtime Crash / Init Error"
description: "Diagnose Lambda runtime crashes during initialization or execution"
status: active
severity: CRITICAL
triggers:
  - "Runtime exited with error"
  - "Init error"
  - "RequestId.*Error"
  - "Runtime.ExitError"
  - "exit status"
owner: devops-agent
objective: "Identify the runtime crash cause and restore function execution"
context: "Runtime crashes differ from unhandled exceptions — the runtime process itself dies. Causes: native library incompatibility, segfault, corrupted deployment package, incompatible runtime version, or init code that calls process.exit/os.exit."
---

## Phase 1 — Triage

MUST:
- Check CloudWatch logs for "Runtime exited with error" and exit status code
- Check function runtime: `aws lambda get-function-configuration` → Runtime
- Check if error occurs during Init phase (before handler) or during Invoke phase
- Verify deployment package architecture matches Lambda (x86_64 vs arm64)

SHOULD:
- Check if native libraries (C extensions, shared objects) are compiled for Amazon Linux 2
- Check if runtime version is supported (not deprecated)

## Common Issues

- symptoms: "Runtime exited with error: exit status 1 during Init"
  diagnosis: "Init code (global scope) is crashing. Import error, missing native lib, or explicit exit."
  resolution: "Check global scope code. Ensure native libs are compiled for Amazon Linux 2 / Lambda runtime."

- symptoms: "Runtime exited with error: signal: segmentation fault"
  diagnosis: "Native library crash. Binary incompatibility with Lambda environment."
  resolution: "Recompile native dependencies for Amazon Linux 2. Use Lambda-compatible build environment."

- symptoms: "Error after changing runtime version"
  diagnosis: "Code incompatible with new runtime version (deprecated APIs, syntax changes)."
  resolution: "Check runtime changelog. Update code for compatibility. Test locally with SAM/Docker."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
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
2. If a new version was published, point the alias back to the previous version: `aws lambda update-alias --function-name <name> --name <alias> --function-version <previous>`
3. If environment variables were changed, restore previous values: `aws lambda update-function-configuration --function-name <name> --environment Variables=<original>`
4. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<native_lib|init_crash|architecture_mismatch|runtime_version|segfault>"
evidence:
  - type: cloudwatch_logs
    content: "<runtime error message and exit code>"
severity: CRITICAL
mitigation:
  immediate: "Fix deployment package or revert runtime version"
  long_term: "Use Lambda-compatible build pipeline, test with SAM local"
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
