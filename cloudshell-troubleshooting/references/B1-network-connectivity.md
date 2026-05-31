---
title: "B1 — Network Connectivity"
description: "Diagnose CloudShell network connectivity issues"
status: active
severity: HIGH
triggers: ["no internet", "cannot reach endpoint", "connection refused", "DNS resolution failed"]
owner: devops-agent
objective: "Resolve network connectivity issues in CloudShell environments"
context: "Standard CloudShell has internet access. VPC environments require NAT gateway for internet. Corporate firewalls may block WebSocket connections."
---

## Phase 1 — Triage

MUST:
- Determine if environment is standard or VPC-connected: `aws cloudshell describe-environments`
- Test basic connectivity: `curl -s -o /dev/null -w "%{http_code}" https://aws.amazon.com` (inside CloudShell)
- Test DNS resolution: `nslookup sts.amazonaws.com` (inside CloudShell)

SHOULD:
- Check if specific AWS service endpoints are reachable: `aws sts get-caller-identity`
- For VPC environments, verify subnet route table: `aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=<subnet-id>"`

MAY:
- Test connectivity to specific ports: `nc -zv <host> <port>` (inside CloudShell)
- Check VPC flow logs for denied traffic: `aws logs filter-log-events --log-group-name <vpc-flow-log-group> --filter-pattern "REJECT"`

## Phase 2 — Remediate

MUST:
- For VPC environments without internet: add NAT gateway to subnet route table
- Verify security group allows outbound traffic: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Ensure NACL allows outbound traffic: `aws ec2 describe-network-acls --filters "Name=association.subnet-id,Values=<subnet-id>"`

SHOULD:
- Add VPC endpoints for frequently used AWS services to avoid internet dependency
- Configure DNS resolution in VPC: `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsSupport`

MAY:
- Set up VPC peering or Transit Gateway for cross-VPC connectivity
- Configure proxy settings in CloudShell if behind corporate proxy

## Safety Ratings

```
safety_ratings:
  - "describe-route-tables / describe-security-groups: GREEN — read-only network inspection"
  - "create-nat-gateway: YELLOW — creates billable resource, modifies network topology"
  - "authorize-security-group-egress: YELLOW — modifies network access rules"
  - "create-vpc-endpoint: YELLOW — creates billable resource, modifies VPC routing"
```

## Escalation Conditions

- VPC environment has no NAT gateway and adding one requires VPC architecture changes
- Security group or NACL changes require network team approval
- DNS resolution failures point to VPC DNS configuration issues
- Corporate firewall blocking CloudShell WebSocket connections

## Data Sensitivity

- `aws ec2 describe-route-tables`: LOW — network configuration metadata
- `aws ec2 describe-security-groups`: MEDIUM — reveals network access rules
- `aws logs filter-log-events` (VPC flow logs): MEDIUM — reveals IP addresses and traffic patterns
- `nslookup / curl`: LOW — basic connectivity tests

## Prohibited Actions

- NEVER open security group to 0.0.0.0/0 for all ports to fix connectivity
- NEVER disable NACLs entirely to troubleshoot
- NEVER modify production VPC route tables without change management approval
- NEVER expose VPC endpoints to public access

## Phase 3 — Rollback

1. If NAT gateway was created in error: `aws ec2 delete-nat-gateway --nat-gateway-id <id>` and remove route table entry
2. If security group rules were modified: `aws ec2 revoke-security-group-egress --group-id <sg-id> --ip-permissions <original-rules>`
3. If VPC endpoint was created unnecessarily: `aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <id>`
4. Restore original route table entries if modified
5. Document network changes and notify the network team

## Common Issues

- symptoms: "curl: (6) Could not resolve host in VPC CloudShell environment"
  diagnosis: "VPC DNS resolution is disabled or DNS server is unreachable."
  resolution: "Enable DNS support on VPC: aws ec2 modify-vpc-attribute --vpc-id <vpc-id> --enable-dns-support."

## Output Format

```yaml
root_cause: "network_connectivity — <specific_cause>"
evidence:
  - type: connectivity_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Restore network connectivity"
  long_term: "Configure VPC endpoints and proper routing for CloudShell VPC environments"
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
