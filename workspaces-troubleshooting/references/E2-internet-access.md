---
title: "E2 — Internet Access"
description: "Diagnose internet access issues from WorkSpaces"
status: active
severity: MEDIUM
triggers:
  - "workspace no internet"
  - "cannot browse web from workspace"
  - "workspace internet blocked"
  - "NAT gateway issue"
owner: devops-agent
objective: "Resolve internet connectivity issues from WorkSpaces"
context: "WorkSpaces in private subnets need a NAT gateway for internet access. WorkSpaces in public subnets can use an Elastic IP. Internet access can also be provided through on-premises network via VPN/Direct Connect."
---

## Phase 1 — Triage

MUST:
- Check subnet route table for internet route: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id> --query 'RouteTables[].Routes[?DestinationCidrBlock==\`0.0.0.0/0\`]'`
- Verify NAT gateway status: `aws ec2 describe-nat-gateways --filter Name=subnet-id,Values=<public-subnet-id> --query 'NatGateways[].{State:State,SubnetId:SubnetId}'`
- Check security group outbound rules: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[].IpPermissionsEgress'`
- Verify NACLs allow outbound internet traffic: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>`

SHOULD:
- Check if internet access is intentionally blocked by policy
- Verify DNS resolution works from the WorkSpace subnet
- Check NAT gateway CloudWatch metrics for errors: `aws cloudwatch get-metric-statistics --namespace AWS/NATGateway --metric-name ErrorPortAllocation --dimensions Name=NatGatewayId,Value=<nat-id> --start-time <iso> --end-time <iso> --period 300 --statistics Sum`

MAY:
- Check if proxy server configuration is required
- Verify on-premises internet breakout if using VPN/DX

## Phase 2 — Remediate

MUST:
- Add NAT gateway in public subnet and route 0.0.0.0/0 through it for private subnets
- Ensure security group allows outbound traffic on required ports (80, 443)
- Fix NACL rules if blocking outbound traffic

SHOULD:
- Use NAT gateway in each AZ for high availability
- Configure proxy settings if corporate policy requires it

MAY:
- Consider AWS Network Firewall for controlled internet access
- Implement URL filtering for security compliance

## Common Issues

- symptoms: "No internet access from WorkSpace"
  diagnosis: "Private subnet missing NAT gateway route or NAT gateway not in available state."
  resolution: "Create NAT gateway in public subnet. Add 0.0.0.0/0 route to NAT gateway in private subnet route table."

- symptoms: "Internet works intermittently"
  diagnosis: "NAT gateway port allocation exhaustion or single-AZ NAT gateway failure."
  resolution: "Check NAT gateway metrics. Deploy NAT gateways in multiple AZs."

- symptoms: "Can reach some sites but not others"
  diagnosis: "Security group or NACL blocking specific ports, or DNS filtering in place."
  resolution: "Check outbound rules for port restrictions. Verify DNS resolution for blocked domains."

## Output Format

```yaml
root_cause: "internet_access — <specific_cause>"
evidence:
  - type: route_table
    content: "<route table entries for internet access>"
  - type: nat_gateway
    content: "<NAT gateway state and metrics>"
severity: MEDIUM
mitigation:
  immediate: "Fix routing, NAT gateway, or security group rules"
  long_term: "Deploy multi-AZ NAT gateways and monitor connectivity"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- WorkSpaces contain user desktop data and files
- Directory integration reveals Active Directory structure
- Connection logs contain user access patterns
- Classification: CONFIDENTIAL - user workspace data and directory information

## Prohibited Actions

- NEVER terminate or rebuild WorkSpaces without user data backup confirmation
- NEVER modify directory service configurations during business hours
- NEVER change MFA settings without user notification and transition period
- NEVER modify VPC or subnet configurations for running WorkSpaces without maintenance window

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
