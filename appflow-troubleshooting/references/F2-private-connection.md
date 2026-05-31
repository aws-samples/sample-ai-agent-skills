---
title: "F2 — Private Connection"
description: "Diagnose Amazon AppFlow private connection and PrivateLink issues"
status: active
severity: HIGH
triggers: ["private connection error", "PrivateLink failed", "VPC endpoint error", "private connectivity"]
owner: devops-agent
objective: "Resolve private connection issues including VPC endpoints, security groups, DNS, and connectivity"
context: "Private connections use AWS PrivateLink VPC endpoints. Requires endpoint in AVAILABLE state, correct security groups, and DNS resolution."
---

## Phase 1 — Triage

MUST:
- Check VPC endpoint status: `aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.<region>.appflow --query 'VpcEndpoints[*].{Id:VpcEndpointId,State:State,SubnetIds:SubnetIds}'`
- Verify security group allows required traffic: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Check private connection in AppFlow: `aws appflow describe-connector-profiles --connector-profile-names <profile>`

SHOULD:
- Verify DNS resolution for the private endpoint
- Check VPC route tables for correct routing

MAY:
- Test connectivity from a VPC instance to the private endpoint
- Review VPC Flow Logs for denied traffic: `aws ec2 describe-flow-logs --filter Name=resource-id,Values=<vpc-id>`

## Phase 2 — Remediate

MUST:
- Ensure VPC endpoint is in AVAILABLE state
- Fix security group to allow inbound HTTPS (443) from AppFlow
- Verify DNS resolution returns private IP addresses

SHOULD:
- Enable private DNS for the VPC endpoint
- Verify subnet has available IP addresses

MAY:
- Configure VPC endpoint policy for least-privilege access

## Safety Ratings

```
safety_ratings:
  - "ec2 describe-vpc-endpoints: GREEN — read-only endpoint status"
  - "ec2 describe-security-groups: GREEN — read-only security group rules"
  - "ec2 authorize-security-group-ingress: YELLOW — modifies network access"
  - "ec2 create-vpc-endpoint: YELLOW — creates network infrastructure"
```

## Escalation Conditions

- VPC endpoint creation requires networking team approval
- Security group changes affect other services in the VPC
- DNS resolution issues require Route 53 or DHCP option set changes
- Private connection requires cross-account VPC peering

## Data Sensitivity

- `aws ec2 describe-vpc-endpoints`: MEDIUM — reveals VPC networking configuration
- `aws ec2 describe-security-groups`: MEDIUM — reveals network access rules
- VPC Flow Logs: HIGH — reveal network traffic patterns and IP addresses

## Prohibited Actions

- NEVER open security groups to 0.0.0.0/0 to troubleshoot private connections
- NEVER delete VPC endpoints that are used by other services
- NEVER modify VPC route tables without understanding the impact on other workloads
- NEVER disable private DNS without confirming no other services depend on it

## Phase 3 — Rollback

1. If security group rules were added, remove them: `aws ec2 revoke-security-group-ingress --group-id <sg-id> --protocol tcp --port 443 --cidr <cidr>`
2. If a VPC endpoint was created for testing, delete it: `aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <endpoint-id>`
3. Restore original DNS configuration if modified
4. Verify other services in the VPC are not affected
5. Document the correct private connection configuration

## Common Issues

- symptoms: "Private connection shows 'Error' status in AppFlow"
  diagnosis: "VPC endpoint is not in AVAILABLE state or security group blocks traffic."
  resolution: "Verify VPC endpoint state. Ensure security group allows inbound HTTPS (443) from the AppFlow service."

## Output Format

```yaml
root_cause: "private_connection — <specific_cause>"
evidence:
  - type: vpc_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix VPC connectivity"
  long_term: "Monitor endpoint health"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
