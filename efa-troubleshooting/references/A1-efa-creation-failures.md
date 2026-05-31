---
title: "A1 — EFA Creation Failures"
description: "Diagnose EFA network interface creation failures"
status: active
severity: HIGH
triggers: ["efa creation failed", "cannot create efa", "efa not supported", "InvalidParameterValue efa"]
owner: devops-agent
objective: "Resolve EFA network interface creation failures"
context: "EFA interfaces are created as network interfaces with InterfaceType=efa. Only specific instance types support EFA."
---

## Phase 1 — Triage

MUST:
- Verify instance type supports EFA: `aws ec2 describe-instance-types --instance-types <type> --query 'InstanceTypes[0].{EfaSupported:NetworkInfo.EfaSupported,MaxEfa:NetworkInfo.EfaInfo.MaximumEfaInterfaces}'`
- Check existing network interfaces: `aws ec2 describe-network-interfaces --filters "Name=attachment.instance-id,Values=<instance-id>" --query 'NetworkInterfaces[*].{Id:NetworkInterfaceId,Type:InterfaceType}'`
- Verify IAM permissions for ec2:CreateNetworkInterface with EFA

SHOULD:
- Check if the maximum number of EFA interfaces is already reached
- Verify the subnet and availability zone support the instance type

MAY:
- Review CloudTrail for CreateNetworkInterface errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateNetworkInterface --max-results 10`

## Phase 2 — Remediate

MUST:
- Use an EFA-supported instance type
- Create EFA interface with correct parameters: `aws ec2 create-network-interface --subnet-id <subnet> --interface-type efa --groups <sg-id>`
- Ensure security group allows all self-traffic

SHOULD:
- Specify EFA at launch time in the launch template rather than creating separately
- Use placement groups for optimal performance

MAY:
- Automate EFA creation in CloudFormation or Terraform

## Safety Ratings

```
safety_ratings:
  - "ec2 describe-instance-types: GREEN — read-only instance type query"
  - "ec2 describe-network-interfaces: GREEN — read-only interface enumeration"
  - "ec2 create-network-interface: YELLOW — creates a network resource"
  - "ec2 attach-network-interface: YELLOW — modifies instance networking"
```

## Escalation Conditions

- EFA creation fails with internal service errors after verifying all parameters
- Instance type is listed as EFA-supported but creation still fails
- EFA interface limit reached and additional interfaces are required
- Subnet or AZ capacity prevents EFA-supported instance launch

## Data Sensitivity

- `aws ec2 describe-instance-types`: LOW — returns public instance type specifications
- `aws ec2 describe-network-interfaces`: MEDIUM — reveals VPC topology and security groups
- `aws cloudtrail lookup-events`: MEDIUM — reveals API actions and IAM principals
- Security group IDs: LOW — identifiers only

## Prohibited Actions

- NEVER create EFA interfaces in security groups that allow all inbound from 0.0.0.0/0
- NEVER attempt to create EFA on unsupported instance types repeatedly
- NEVER delete existing network interfaces on running instances to make room for EFA
- NEVER ignore the maximum EFA interface limit for the instance type

## Phase 3 — Rollback

1. Delete incorrectly created EFA interfaces: `aws ec2 delete-network-interface --network-interface-id <eni-id>`
2. If launch template was modified, revert to previous version
3. If security group was created for EFA, delete if no longer needed: `aws ec2 delete-security-group --group-id <sg-id>`
4. Verify no orphaned network interfaces remain in the subnet
5. Document the failure cause and correct instance type requirements

## Common Issues

- symptoms: "CreateNetworkInterface returns InvalidParameterValue for InterfaceType efa"
  diagnosis: "The instance type does not support EFA or the subnet is in an unsupported AZ."
  resolution: "Verify EFA support for the instance type. Use describe-instance-types to confirm."

- symptoms: "EFA interface created but cannot be attached to instance"
  diagnosis: "Instance is running (EFA requires stopped state) or maximum interfaces reached."
  resolution: "Stop the instance before attaching EFA, or specify EFA at launch time."

## Output Format

```yaml
root_cause: "efa_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix instance type or creation parameters"
  long_term: "Use launch templates with EFA pre-configured"
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
