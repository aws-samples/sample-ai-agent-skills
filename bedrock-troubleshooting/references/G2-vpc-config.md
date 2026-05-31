---
title: "G2 — VPC Configuration Issues"
description: "Diagnose VPC endpoint configuration for Bedrock"
status: active
severity: HIGH
triggers:
  - "VPC endpoint Bedrock"
  - "private access Bedrock"
  - "cannot reach Bedrock from VPC"
  - "VPC connectivity"
owner: devops-agent
objective: "Identify and resolve VPC configuration issues for Bedrock access"
context: "Bedrock API calls from private subnets require VPC interface endpoints. Both bedrock (control plane) and bedrock-runtime (data plane) endpoints may be needed. Security groups must allow HTTPS. Private DNS must be enabled."
---

## Phase 1 — Triage

MUST:
- Check for Bedrock VPC endpoints: `aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=<vpc-id> Name=service-name,Values=com.amazonaws.*.bedrock*`
- Verify endpoint security groups allow HTTPS (443)
- Check VPC DNS settings (enableDnsSupport, enableDnsHostnames)
- Verify private DNS is enabled on the endpoint

SHOULD:
- Check if both bedrock and bedrock-runtime endpoints exist
- Verify endpoint subnet placement
- Check endpoint policy (default allows all)
- Test connectivity from within the VPC

MAY:
- Check for bedrock-agent-runtime endpoint (for agent/KB operations)
- Verify route table configuration

## Phase 2 — Remediate

MUST:
- Create VPC interface endpoints for bedrock-runtime (and bedrock if needed)
- Configure security groups to allow HTTPS from client resources
- Enable private DNS on the VPC and endpoints

SHOULD:
- Create endpoints in multiple AZs for availability
- Use default endpoint policy (allow all) unless restrictions needed
- Test Bedrock API calls from within the VPC

MAY:
- Create VPC endpoint templates for Bedrock
- Implement endpoint monitoring

## Common Issues

- symptoms: "Timeout when calling Bedrock from private subnet"
  diagnosis: "No VPC endpoint for Bedrock or security group blocking."
  resolution: "Create bedrock-runtime VPC endpoint. Allow HTTPS in security group."

- symptoms: "Control plane calls work but invocations fail"
  diagnosis: "bedrock endpoint exists but bedrock-runtime endpoint missing."
  resolution: "Create bedrock-runtime VPC interface endpoint."

## Output Format

```yaml
root_cause: "vpc_config — <specific_cause>"
evidence:
  - type: vpc_endpoints
    content: "<VPC endpoint configuration>"
  - type: security_groups
    content: "<endpoint security group rules>"
severity: HIGH
mitigation:
  immediate: "Create required VPC endpoints"
  long_term: "Standardize VPC configuration for Bedrock access"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Create VPC interface endpoints for bedrock-runtime | YELLOW |
| Configure security groups to allow HTTPS from client resources | YELLOW |
| Enable private DNS on the VPC and endpoints | YELLOW |
| Create endpoints in multiple AZs for availability | GREEN |
| Use default endpoint policy (allow all) unless restrictions needed | YELLOW |
| Test Bedrock API calls from within the VPC | GREEN |
| Create VPC endpoint templates for Bedrock | GREEN |
| Implement endpoint monitoring | GREEN |

## Escalation Conditions

- Fix involves modifying guardrails on production model
- Knowledge base contains sensitive data

## Data Sensitivity

- **HIGH**: model invocation data, knowledge base content
- VPC endpoint configurations reveal network architecture and access patterns

## Prohibited Actions

- NEVER suggest removing guardrails to fix content filtering
- NEVER suggest using production data for fine-tuning without approval

## Phase 3 — Rollback

1. If VPC endpoints were created, delete them: `aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <endpoint-id>`
2. If security group rules were added, remove the added rules
3. If private DNS was enabled, disable it on the VPC endpoint if it causes DNS conflicts
4. Verify rollback by confirming Bedrock access returns to the previous connectivity method

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
  - "NEVER suggest removing guardrails from production models"
  - "NEVER suggest disabling content filtering"
  - "NEVER suggest overly broad model access permissions"
