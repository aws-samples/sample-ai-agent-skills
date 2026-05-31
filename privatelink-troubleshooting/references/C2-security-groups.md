---
title: "C2 — Security Groups"
description: "Diagnose security group issues affecting VPC interface endpoint connectivity"
status: active
severity: HIGH
triggers:
  - "security group blocking"
  - "endpoint security group"
  - "ENI security group"
  - "inbound rule missing"
  - "traffic blocked by security group"
owner: devops-agent
objective: "Resolve security group misconfigurations on interface endpoint ENIs"
context: "Interface endpoints create ENIs with security groups. Inbound rules on endpoint ENIs must allow traffic from clients on the service port. The default VPC security group only allows traffic from the same security group. Both client outbound and endpoint inbound rules must permit the connection. Security groups are stateful."
---

## Phase 1 — Triage

MUST:
- Identify endpoint ENIs and their security groups: `aws ec2 describe-network-interfaces --filters Name=vpc-endpoint-id,Values=<endpoint-id> --query 'NetworkInterfaces[*].{Id:NetworkInterfaceId,SubnetId:SubnetId,Groups:Groups}'`
- Check endpoint security group inbound rules: `aws ec2 describe-security-groups --group-ids <endpoint-sg-id> --query 'SecurityGroups[*].IpPermissions'`
- Check client security group outbound rules: `aws ec2 describe-security-groups --group-ids <client-sg-id> --query 'SecurityGroups[*].IpPermissionsEgress'`
- Identify the service port (typically 443 for HTTPS)

SHOULD:
- Check if the default VPC security group is being used (may be too restrictive)
- Verify security group references (sg-to-sg rules) are correct
- Check VPC Flow Logs for REJECT entries: filter by endpoint ENI IDs

MAY:
- Audit all security groups associated with the endpoint
- Compare security group rules across AZs

## Phase 2 — Remediate

MUST:
- Add inbound rule to endpoint security group: allow TCP on service port from client CIDR or security group
- Verify client security group allows outbound to the endpoint (default allows all outbound)

SHOULD:
- Use security group references instead of CIDR blocks for better maintainability
- Apply consistent security groups across all endpoint ENIs
- Document required security group rules for each endpoint

MAY:
- Implement security group management via IaC (CloudFormation/Terraform)
- Set up Config rules to detect security group drift

## Common Issues

- symptoms: "Default security group blocking endpoint traffic"
  diagnosis: "Default VPC security group only allows inbound from same security group."
  resolution: "Create a dedicated security group for the endpoint with appropriate inbound rules."

- symptoms: "Security group allows CIDR but not all clients can connect"
  diagnosis: "Client CIDR not fully covered by the security group rule."
  resolution: "Use the VPC CIDR or client security group reference instead of specific IP ranges."

- symptoms: "Endpoint works on port 443 but not on custom port"
  diagnosis: "Security group inbound rule only allows port 443."
  resolution: "Add inbound rule for the custom port used by the endpoint service."

## Output Format

```yaml
root_cause: "security_group — <specific_cause>"
evidence:
  - type: endpoint_sg_rules
    content: "<endpoint security group inbound rules>"
  - type: client_sg_rules
    content: "<client security group outbound rules>"
  - type: flow_logs
    content: "<VPC Flow Log REJECT entries>"
severity: HIGH
mitigation:
  immediate: "Fix security group rules to allow required traffic"
  long_term: "Implement security group management via IaC"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: MEDIUM — security groups control network access to endpoints
- Service Disruption Risk: HIGH — incorrect rules block all endpoint traffic
- Compliance Impact: HIGH — security groups are a primary network security control

## Escalation Conditions
- Security group changes block all endpoint traffic
- Overly permissive rules detected on endpoint security groups
- Security group rule conflicts cause intermittent connectivity
- Compliance audit flags security group violations
- Security group changes affect multiple endpoints

## Data Sensitivity
- Security group rules — INTERNAL
- Allowed IP ranges — INTERNAL
- Port and protocol configurations — INTERNAL
- Cross-security-group references — INTERNAL
- Never expose security group details externally

## Prohibited Actions
- Do NOT allow inbound 0.0.0.0/0 on endpoint security groups
- Do NOT remove required security group rules without testing
- Do NOT modify security groups during production hours without approval
- Do NOT create overly broad rules
- Do NOT bypass security group review processes
- Do NOT delete security groups attached to active endpoints

## Phase 3 — Rollback
- Restore previous security group rules
- Re-create deleted security groups
- Revert rule changes that caused connectivity issues
- Verify endpoint connectivity after changes
- Document security group changes
- Update security group management procedures

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
