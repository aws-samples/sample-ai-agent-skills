---
title: "D2 — Canary VPC Issues"
description: "Diagnose Synthetics canary VPC connectivity problems"
status: active
severity: HIGH
triggers:
  - "canary VPC"
  - "canary network"
  - "canary timeout VPC"
  - "canary cannot reach"
  - "canary private endpoint"
owner: devops-agent
objective: "Identify why a VPC-configured canary cannot reach its target endpoint"
context: "Canaries can be configured to run inside a VPC to test private endpoints. When in a VPC, the canary Lambda function needs proper subnet, security group, NAT gateway (for internet), and VPC endpoints (for AWS services). This mirrors Lambda VPC networking behavior."
---

## Phase 1 — Triage

MUST:
- Check canary VPC configuration: `aws synthetics describe-canaries --names <canary-name> --query 'Canaries[].VpcConfig'`
- Verify subnets have route to the target (NAT gateway for internet, VPC endpoint for AWS services)
- Check security group allows outbound traffic to the target endpoint port
- Verify the canary execution role has `ec2:CreateNetworkInterface`, `ec2:DescribeNetworkInterfaces`, `ec2:DeleteNetworkInterface` permissions

SHOULD:
- Check if subnets are private (canaries in public subnets don't get public IPs)
- Verify NAT gateway is in a public subnet with an internet gateway route
- Check VPC endpoint for S3 (canary needs to write artifacts to S3)

MAY:
- Test connectivity from an EC2 instance in the same subnet
- Check ENI limits in the subnets (Lambda creates ENIs for VPC access)

## Phase 2 — Remediate

MUST:
- Place canary in private subnets with NAT gateway for internet access
- Fix security group outbound rules to allow traffic to the target
- Add EC2 network interface permissions to the execution role

SHOULD:
- Add VPC endpoints for S3 and CloudWatch Logs to avoid NAT gateway costs
- Use multiple subnets across AZs for resilience
- Ensure sufficient IP addresses in the subnets

MAY:
- Create a dedicated security group for canaries
- Document the VPC networking requirements for canary deployments

## Common Issues

- symptoms: "Canary times out when VPC is configured"
  diagnosis: "Subnet has no route to the target (missing NAT gateway or VPC endpoint)."
  resolution: "Add NAT gateway for internet targets or VPC endpoint for AWS service targets."

- symptoms: "Canary fails with network error in VPC"
  diagnosis: "Security group does not allow outbound traffic to the target port."
  resolution: "Add outbound rule for the target port (443 for HTTPS, custom ports as needed)."

- symptoms: "Canary fails to store artifacts"
  diagnosis: "No route to S3 from the VPC (missing S3 VPC endpoint or NAT gateway)."
  resolution: "Add S3 gateway VPC endpoint or ensure NAT gateway is configured."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Place canary in private subnets with NAT | GREEN | Network configuration |
| Fix security group outbound rules | YELLOW | Security group changes affect all resources using it |
| Add EC2 network interface permissions | YELLOW | IAM changes |
| Add VPC endpoints for S3 and CW Logs | GREEN | Additive network configuration |

## Escalation Conditions

- VPC canary monitoring broken for production endpoints — escalate to operations team
- Security group changes in production VPCs — escalate to network security team
- NAT gateway or VPC endpoint creation — escalate to network team
- ENI limits reached in subnets — escalate to infrastructure team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws synthetics describe-canaries` (VpcConfig) | MEDIUM | Exposes VPC, subnet, and security group IDs |
| Route table and NAT gateway checks | MEDIUM | Exposes network architecture |

## Prohibited Actions

- **NEVER** suggest deleting log groups with retention data without confirming data is no longer needed
- **NEVER** suggest disabling alarms in production
- **NEVER** suggest removing metric filters that feed production alarms
- **NEVER** open overly broad security group rules (e.g., 0.0.0.0/0 on all ports)

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Changed security group rules | Revert security group rules |
| Added VPC endpoints | Delete: `aws ec2 delete-vpc-endpoints` |
| Changed canary VPC configuration | Revert canary VPC config |

## Output Format

```yaml
root_cause: "canary_vpc — <specific_cause>"
evidence:
  - type: vpc_config
    content: "<canary VPC configuration>"
  - type: routing
    content: "<subnet route table and NAT/VPC endpoint status>"
severity: HIGH
mitigation:
  immediate: "Fix routing, security groups, or IAM permissions"
  long_term: "Document VPC requirements for canary deployments"
```

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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
