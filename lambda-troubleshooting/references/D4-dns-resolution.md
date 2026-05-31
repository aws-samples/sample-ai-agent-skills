---
title: "D4 — Lambda DNS Resolution Failures"
description: "Diagnose DNS resolution failures in VPC Lambda"
status: active
severity: MEDIUM
triggers:
  - "getaddrinfo ENOTFOUND"
  - "Name resolution failed"
  - "DNS.*timeout"
owner: devops-agent
objective: "Restore DNS resolution for VPC Lambda functions"
context: "VPC Lambda uses the VPC's DNS resolver (CIDR+2). Requires enableDnsSupport=true. Private hosted zones require enableDnsHostnames=true. Custom DHCP options can override DNS."
---

## Common Issues

- symptoms: "getaddrinfo ENOTFOUND for AWS service endpoints"
  diagnosis: "VPC DNS not enabled, or DHCP options pointing to unreachable DNS."
  resolution: "Enable enableDnsSupport on VPC. Check DHCP options set."

- symptoms: "Private hosted zone names don't resolve"
  diagnosis: "enableDnsHostnames not enabled, or hosted zone not associated with VPC."
  resolution: "Enable enableDnsHostnames. Associate hosted zone with VPC."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
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
2. If VPC configuration was changed, restore original VPC settings: `aws lambda update-function-configuration --function-name <name> --vpc-config SubnetIds=<original>,SecurityGroupIds=<original>`
3. If event source mapping was modified, restore original configuration: `aws lambda update-event-source-mapping --uuid <uuid> --batch-size <original>`
## Output Format

```yaml
root_cause: "<dns_support_disabled|dhcp_misconfigured|hosted_zone_not_associated>"
severity: MEDIUM
mitigation:
  immediate: "Enable VPC DNS settings"
  long_term: "Use VPC endpoints with private DNS for AWS services"
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
