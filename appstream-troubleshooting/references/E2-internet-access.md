---
title: "E2 — Internet Access Issues"
description: "Diagnose internet access problems from AppStream streaming sessions"
status: active
severity: MEDIUM
triggers:
  - "no internet access"
  - "cannot browse internet"
  - "internet blocked"
  - "web access from session"
owner: devops-agent
objective: "Identify and resolve internet access issues from AppStream sessions"
context: "Internet access from AppStream depends on fleet configuration. Non-VPC fleets have internet access by default. VPC fleets need NAT gateway or internet gateway for internet access. Security groups and NACLs control outbound traffic."
---

## Phase 1 — Triage

MUST:
- Check if the fleet uses a VPC: `aws appstream describe-fleets --names <fleet-name> --query 'Fleets[*].VpcConfig'`
- If VPC, check for NAT gateway: `aws ec2 describe-nat-gateways --filter Name=vpc-id,Values=<vpc-id>`
- Check security group outbound rules: `aws ec2 describe-security-groups --group-ids <sg-ids> --query 'SecurityGroups[*].IpPermissionsEgress'`
- Check subnet route tables for internet route: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`

SHOULD:
- Verify NAT gateway is in a public subnet with internet gateway
- Check NACLs for outbound traffic restrictions
- Verify DNS resolution works from the fleet

MAY:
- Check if a proxy is configured on the image
- Test with a non-VPC fleet to isolate the issue

## Phase 2 — Remediate

MUST:
- For VPC fleets, set up NAT gateway in a public subnet
- Add route to NAT gateway in the fleet subnet's route table: `0.0.0.0/0 → nat-gw-id`
- Allow outbound traffic in security groups (at minimum HTTPS 443)

SHOULD:
- Allow outbound DNS (UDP 53) in security groups
- Configure proxy settings on the image if required by corporate policy

MAY:
- Use VPC endpoints for AWS services to reduce NAT gateway traffic
- Set up web filtering if internet access needs to be restricted

## Common Issues

- symptoms: "No internet access from VPC fleet"
  diagnosis: "No NAT gateway or route to NAT gateway."
  resolution: "Create NAT gateway in public subnet. Add 0.0.0.0/0 route to fleet subnet."

- symptoms: "HTTPS works but HTTP doesn't"
  diagnosis: "Security group only allows outbound HTTPS (443)."
  resolution: "Add outbound HTTP (80) to security group if needed."

- symptoms: "DNS resolution fails"
  diagnosis: "Security group blocking outbound DNS (UDP 53)."
  resolution: "Allow outbound UDP 53 in the security group."

## Output Format

```yaml
root_cause: "internet_access_issue — <specific_cause>"
evidence:
  - type: vpc_config
    content: "<fleet VPC configuration>"
  - type: nat_gateway
    content: "<NAT gateway status>"
  - type: route_table
    content: "<subnet route table entries>"
severity: MEDIUM
mitigation:
  immediate: "Fix NAT gateway and routing"
  long_term: "Document network architecture for AppStream"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Set up NAT gateway in public subnet | YELLOW | Network infrastructure change |
| Add route to NAT gateway | YELLOW | Route table change |
| Allow outbound traffic in security groups | YELLOW | Security group change |
| Allow outbound DNS (UDP 53) | YELLOW | Security group change |
| Configure proxy settings on image | YELLOW | Image configuration change |

## Escalation Conditions

- Fix requires fleet rebuild affecting active users

## Data Sensitivity

- **NAT gateway**: Network address translation configuration — INTERNAL
- **Route tables**: Subnet routing configuration — INTERNAL
- **Security group rules**: Outbound access rules — INTERNAL

## Prohibited Actions

- NEVER suggest deleting fleet with active sessions
- NEVER allow unrestricted internet access without security review
- NEVER modify routing on subnets used by active fleets without testing

## Phase 3 — Rollback

1. Revert route table changes (remove NAT gateway route)
2. Revert security group outbound rule changes
3. Restore previous proxy settings on the image if modified
4. Verify fleet instances still have required connectivity
5. Test internet access from a session after rollback

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
