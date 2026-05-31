---
title: "D1 — VPC Lambda No Internet Access"
description: "Diagnose VPC-attached Lambda functions that cannot reach the internet or AWS services"
status: active
severity: HIGH
triggers:
  - "connect ETIMEDOUT"
  - "getaddrinfo.*ENOTFOUND"
  - "Task timed out"
  - "VPC.*no internet"
owner: devops-agent
objective: "Restore internet/AWS service connectivity for VPC Lambda"
context: "VPC Lambda functions are placed in private subnets. They do NOT get public IPs. Internet access requires NAT gateway. AWS service access requires NAT gateway or VPC endpoints. This is the #1 VPC Lambda issue."
---

## Phase 1 — Triage

MUST:
- Confirm function is in VPC: `aws lambda get-function-configuration` → VpcConfig
- Check subnet route tables: must have 0.0.0.0/0 → NAT gateway for internet
- Check security group outbound rules: must allow HTTPS (443) outbound
- Check if NAT gateway exists and is in a public subnet with IGW route

SHOULD:
- Check if VPC endpoints exist for the AWS services being called (S3, DynamoDB, SQS, etc.)
- Verify DNS resolution works (enableDnsSupport on VPC)

## Common Issues

- symptoms: "Function times out calling AWS APIs (S3, DynamoDB, SQS)"
  diagnosis: "VPC Lambda has no NAT gateway or VPC endpoint for the target service."
  resolution: "Add NAT gateway or VPC endpoints. Gateway endpoints (S3, DynamoDB) are free."

- symptoms: "Function times out calling external APIs"
  diagnosis: "No NAT gateway for internet access."
  resolution: "Add NAT gateway in public subnet. Route 0.0.0.0/0 → NAT in Lambda's subnet."

- symptoms: "Function works sometimes, fails sometimes"
  diagnosis: "Lambda subnets span multiple AZs. NAT gateway may only be in one AZ."
  resolution: "Ensure NAT gateway (or route) exists in every AZ where Lambda has subnets."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Network configuration changes: YELLOW - May affect connectivity"
  - "Certificate/TLS changes: RED - May cause downtime if misconfigured"
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
root_cause: "<no_nat|no_vpc_endpoint|sg_blocking|route_missing>"
evidence:
  - type: vpc_config
    content: "<subnet route tables and NAT gateway state>"
severity: HIGH
mitigation:
  immediate: "Add NAT gateway or VPC endpoints"
  long_term: "Use VPC endpoints for AWS services (cheaper than NAT), review if VPC is needed"
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
