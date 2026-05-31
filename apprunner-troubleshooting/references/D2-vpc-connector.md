---
title: "D2 — App Runner VPC Connector Issues"
description: "Diagnose VPC connector configuration and connectivity problems"
status: active
severity: HIGH
triggers:
  - "VPC connector"
  - "private subnet"
  - "cannot connect to RDS"
  - "cannot connect to ElastiCache"
  - "VPC access"
  - "private resource"
owner: devops-agent
objective: "Identify and fix VPC connector configuration issues preventing access to private resources"
context: "App Runner services run outside your VPC by default. A VPC connector must be attached to access private resources like RDS, ElastiCache, or internal ALBs. The VPC connector specifies subnets and security groups. Common issues include incorrect subnet selection, security group rules blocking traffic, missing route table entries, and DNS resolution failures for private endpoints."
---

## Phase 1 — Triage

MUST:
- Check if a VPC connector is attached: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.NetworkConfiguration'`
- Describe the VPC connector: `aws apprunner describe-vpc-connector --vpc-connector-arn <vpc-connector-arn>`
- Check security group rules: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Check subnet route tables: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`
- Verify the target resource's security group allows inbound from the VPC connector security group

SHOULD:
- Verify subnets have available IP addresses: `aws ec2 describe-subnets --subnet-ids <subnet-ids> --query 'Subnets[*].{Id:SubnetId,AZ:AvailabilityZone,AvailableIPs:AvailableIpAddressCount}'`
- Check if the target resource is in the same VPC as the VPC connector subnets
- Verify DNS resolution works for the target endpoint

MAY:
- Check VPC flow logs for rejected traffic
- Verify NAT gateway configuration if the application also needs internet access through the VPC
- List all VPC connectors: `aws apprunner list-vpc-connectors`

## Phase 2 — Remediate

MUST:
- Create a VPC connector if none exists: specify subnets in the same VPC as the target resource and a security group that allows outbound traffic
- Ensure the target resource's security group allows inbound traffic from the VPC connector's security group
- Use private subnets with a NAT gateway if the application also needs internet access through the VPC path

SHOULD:
- Use subnets in multiple AZs for high availability
- Create a dedicated security group for the VPC connector with least-privilege rules
- Verify the VPC connector subnets can reach the target resource (same VPC, peering, or Transit Gateway)

MAY:
- Use VPC endpoints for AWS services (S3, DynamoDB, SQS) to avoid NAT gateway costs
- Enable VPC flow logs for ongoing connectivity monitoring

## Common Issues

- symptoms: "Application cannot connect to RDS — connection timeout"
  diagnosis: "No VPC connector attached, or VPC connector subnets are in a different VPC than RDS."
  resolution: "Attach a VPC connector with subnets in the same VPC as RDS. Ensure security groups allow traffic."

- symptoms: "VPC connector attached but connection still fails"
  diagnosis: "Target resource security group does not allow inbound from the VPC connector security group."
  resolution: "Add an inbound rule to the target's security group allowing traffic from the VPC connector's security group."

- symptoms: "Application loses internet access after attaching VPC connector"
  diagnosis: "VPC-routed traffic goes through the VPC. If subnets lack a NAT gateway, internet access via VPC fails."
  resolution: "Use subnets with a NAT gateway, or use VPC endpoints for AWS services."

## Output Format

```yaml
root_cause: "vpc_connector — <specific_cause>"
evidence:
  - type: vpc_connector_config
    content: "<subnets, security groups, VPC>"
  - type: security_group_rules
    content: "<inbound/outbound rules>"
  - type: route_table
    content: "<route table entries>"
severity: HIGH
mitigation:
  immediate: "Fix VPC connector configuration, security groups, or routing"
  long_term: "Document VPC architecture and implement connectivity monitoring"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Create VPC connector | YELLOW | New networking resource |
| Ensure target security group allows inbound | YELLOW | Security group rule change |
| Use private subnets with NAT gateway | YELLOW | Network architecture change |
| Use subnets in multiple AZs | GREEN | Best practice |
| Create dedicated security group | YELLOW | New security group |
| Use VPC endpoints | YELLOW | New VPC endpoint creation |

## Escalation Conditions

- Fix requires redeployment of production service
- NEVER suggest scaling to 0 (not supported)

## Data Sensitivity

- **VPC connector config**: Subnets, security groups — INTERNAL
- **Security group rules**: Inbound/outbound rules — INTERNAL
- **Route tables**: Network routing configuration — INTERNAL

## Prohibited Actions

- NEVER suggest hardcoding credentials in environment variables
- NEVER suggest scaling to 0 instances (not supported by App Runner)
- NEVER open security groups to 0.0.0.0/0 for private resource access

## Phase 3 — Rollback

1. Detach VPC connector from the service: update-service with no VPC connector
2. Revert security group rule changes on target resources
3. Delete the VPC connector if no longer needed: `aws apprunner delete-vpc-connector --vpc-connector-arn <arn>`
4. Verify the service can still reach public endpoints
5. Verify private resource access is no longer needed or use alternative approach

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
