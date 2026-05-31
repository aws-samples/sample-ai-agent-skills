---
title: "A1 — Flow Creation Failures"
description: "Diagnose MediaConnect flow creation failures"
status: active
severity: HIGH
triggers: ["cannot create flow", "flow creation failed", "CreateFlow error"]
owner: devops-agent
objective: "Resolve flow creation issues and get the flow to STANDBY or ACTIVE status"
context: "Flows require a source configuration, valid protocol settings, and appropriate IAM/network permissions. VPC flows need ENI capacity."
---

## Phase 1 — Triage

MUST:
- Check existing flows and quotas: `aws mediaconnect list-flows --query 'Flows[*].{Name:Name,FlowArn:FlowArn,Status:Status}'`
- Verify IAM permissions for mediaconnect:CreateFlow
- Check service quotas: `aws service-quotas get-service-quota --service-code mediaconnect --quota-code L-A99016A8`

SHOULD:
- Review CloudTrail for CreateFlow errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateFlow --max-results 10`
- Verify the source protocol and port configuration are valid

MAY:
- Check if the AZ supports MediaConnect
- Review VPC subnet and ENI capacity if creating a VPC flow

## Phase 2 — Remediate

MUST:
- Fix IAM permissions — ensure the role has mediaconnect:CreateFlow and ec2:DescribeSubnets (for VPC flows)
- Correct source protocol configuration (valid protocol, port range, CIDR)
- Request quota increase if flow limit reached

SHOULD:
- Use a supported protocol: Zixi-push, Zixi-pull, SRT-listener, SRT-caller, RIST, RTP, RTP-FEC, CDI
- Verify the source whitelist CIDR is correct for the sender IP

MAY:
- Create the flow via CloudFormation or Terraform for repeatability
- Tag the flow for cost allocation and identification

## Safety Ratings

```
safety_ratings:
  - "list-flows: GREEN — read-only enumeration of existing flows"
  - "CreateFlow: YELLOW — creates a new flow resource, incurs cost when active"
  - "service-quotas get-service-quota: GREEN — read-only quota check"
  - "cloudtrail lookup-events: GREEN — read-only audit log query"
```

## Escalation Conditions

- Flow creation fails with InternalServerError after all configuration is verified correct
- Quota increase request is denied and flow limit is reached
- VPC flow creation fails due to ENI limits that cannot be increased
- Flow creation blocked by SCP or permission boundary the operator cannot modify

## Data Sensitivity

- `aws mediaconnect list-flows`: LOW — returns flow metadata only
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principal identities and source IPs
- `aws mediaconnect create-flow`: MEDIUM — contains source IP addresses and encryption keys
- `aws service-quotas get-service-quota`: LOW — returns quota values only

## Prohibited Actions

- NEVER create flows in production regions without explicit approval
- NEVER hardcode encryption keys or passphrases in CLI commands — use Secrets Manager
- NEVER ignore source whitelist CIDR configuration — leaving it open (0.0.0.0/0) is a security risk
- NEVER create VPC flows without verifying security group rules first

## Phase 3 — Rollback

1. If a flow was created in error, stop it first: `aws mediaconnect stop-flow --flow-arn <flow-arn>`
2. Delete the erroneous flow: `aws mediaconnect delete-flow --flow-arn <flow-arn>`
3. If VPC interfaces were created, verify ENIs are released after flow deletion
4. If a quota increase was requested unnecessarily, note it for future reference
5. Document the rollback action and notify the media operations team

## Common Issues

- symptoms: "CreateFlow returns LimitExceededException"
  diagnosis: "Account has reached the maximum number of flows in this region."
  resolution: "Request a quota increase via Service Quotas console or delete unused flows."

- symptoms: "CreateFlow returns BadRequestException with invalid source configuration"
  diagnosis: "Source protocol, port, or CIDR configuration is malformed."
  resolution: "Verify protocol is supported, port is in valid range (1024-65535), and CIDR is valid IPv4 notation."

- symptoms: "CreateFlow returns AccessDeniedException"
  diagnosis: "IAM role or user lacks mediaconnect:CreateFlow permission."
  resolution: "Attach the AWSElementalMediaConnectFullAccess managed policy or add mediaconnect:CreateFlow to the IAM policy."

## Output Format

```yaml
root_cause: "flow_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details from API response>"
severity: HIGH
mitigation:
  immediate: "Fix creation configuration and retry"
  long_term: "Deploy flows via IaC with validated templates"
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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
