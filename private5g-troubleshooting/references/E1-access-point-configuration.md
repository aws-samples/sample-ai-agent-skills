---
title: "E1 — Access Point Configuration"
description: "Diagnose access point and network configuration issues"
status: active
severity: MEDIUM
triggers: ["access point error", "APN configuration", "network configuration wrong"]
owner: devops-agent
objective: "Resolve access point and network configuration issues"
context: "Access point configuration defines how devices connect to the Private 5G network and route traffic to the VPC."
---

## Phase 1 — Triage

MUST:
- Get network details: `aws private-networks get-network --network-arn <arn>`
- List network resources: `aws private-networks list-network-resources --network-arn <arn>`
- Verify network status is AVAILABLE

SHOULD:
- Check VPC subnet configuration associated with the network
- Verify security group rules on the Private 5G ENI: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Review route table for the subnet: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`

MAY:
- Check device APN settings match the network configuration
- Review DNS configuration for the VPC

## Phase 2 — Remediate

MUST:
- Ensure VPC subnet has available IP addresses
- Verify security group allows required traffic (inbound and outbound)
- Confirm route table has proper routes for device traffic

SHOULD:
- Update device APN settings to match network configuration
- Ensure DNS resolution works for devices on the network

MAY:
- Configure VPC flow logs to monitor Private 5G traffic
- Set up CloudWatch alarms for network health metrics

## Safety Ratings

```
safety_ratings:
  - "get-network: GREEN — read-only network details"
  - "describe-security-groups: GREEN — read-only security group review"
  - "describe-route-tables: GREEN — read-only routing review"
  - "modify security group rules: YELLOW — affects network access"
```

## Escalation Conditions

- Network configuration changes do not take effect after 30 minutes
- VPC integration shows errors in CloudTrail
- Security group changes cause unintended access issues
- DNS resolution fails for devices on the Private 5G network

## Data Sensitivity

- `aws private-networks get-network`: LOW — returns network configuration
- `aws ec2 describe-security-groups`: MEDIUM — reveals network access rules
- `aws ec2 describe-route-tables`: MEDIUM — reveals network routing topology

## Prohibited Actions

- NEVER open security groups to 0.0.0.0/0 for troubleshooting
- NEVER modify VPC route tables without understanding the impact on other resources
- NEVER delete the Private 5G ENI
- NEVER change subnet associations without a maintenance window

## Phase 3 — Rollback

1. If security group rules were modified, restore original rules: `aws ec2 revoke-security-group-ingress` / `aws ec2 authorize-security-group-ingress`
2. If route table was modified, restore original routes
3. Verify device connectivity is restored after rollback
4. Confirm other VPC resources are unaffected by changes
5. Document all configuration changes and their effects

## Common Issues

- symptoms: "Devices connect to Private 5G but cannot reach VPC resources"
  diagnosis: "Security group or route table blocking traffic from Private 5G ENI."
  resolution: "Update security group to allow traffic from the Private 5G subnet CIDR. Verify route table entries."

- symptoms: "Device shows connected but DNS resolution fails"
  diagnosis: "VPC DNS settings not configured for Private 5G devices."
  resolution: "Ensure VPC has DNS resolution enabled. Configure DHCP options set with proper DNS servers."

## Output Format

```yaml
root_cause: "access_point_configuration — <specific_cause>"
evidence:
  - type: configuration_details
    content: "<config data>"
severity: MEDIUM
mitigation:
  immediate: "Fix configuration issue"
  long_term: "Document and version control network configuration"
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
