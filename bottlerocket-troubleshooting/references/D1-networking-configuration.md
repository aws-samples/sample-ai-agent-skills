---
title: "D1 — Networking Configuration"
description: "Diagnose Bottlerocket networking, DNS, and proxy configuration issues"
status: active
severity: HIGH
triggers: ["dns resolution failed", "network unreachable", "proxy not working", "cannot reach endpoints", "networking issues"]
owner: devops-agent
objective: "Resolve Bottlerocket networking configuration issues including DNS, proxy, and connectivity"
context: "Bottlerocket network settings are configured through the settings API. Direct edits to /etc/resolv.conf or network files are overwritten."
---

## Phase 1 — Triage

MUST:
- Check network settings: `aws ssm start-session --target <instance-id> --document-name AWS-StartInteractiveCommand --parameters command="apiclient get settings.network"`
- Verify DNS settings: `aws ssm start-session --target <instance-id> --document-name AWS-StartInteractiveCommand --parameters command="apiclient get settings.dns"`
- Check VPC and subnet configuration: `aws ec2 describe-instances --instance-ids <instance-id> --query 'Reservations[0].Instances[0].{VpcId:VpcId,SubnetId:SubnetId,PrivateIp:PrivateIpAddress,PublicIp:PublicIpAddress}'`

SHOULD:
- Verify route table for the subnet: `aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=<subnet-id>" --query 'RouteTables[0].Routes'`
- Check security group rules: `aws ec2 describe-security-groups --group-ids <sg-id>`

MAY:
- Test connectivity from admin container using sheltie and ping/curl

## Phase 2 — Remediate

MUST:
- Configure DNS via settings API: `apiclient set dns.name-servers=["10.0.0.2"]`
- Configure proxy via settings API: `apiclient set network.https-proxy="http://proxy:3128"`
- Set no-proxy list: `apiclient set network.no-proxy=["169.254.169.254","10.0.0.0/8"]`

SHOULD:
- Ensure the VPC DHCP options set provides correct DNS servers
- Verify that proxy settings include all required no-proxy entries for AWS services

MAY:
- Configure custom hostname: `apiclient set network.hostname="custom-hostname"`

## Safety Ratings

```
safety_ratings:
  - "apiclient get settings.network: GREEN — read-only network settings query"
  - "apiclient set dns.name-servers: YELLOW — changes DNS resolution, may break connectivity"
  - "apiclient set network.https-proxy: YELLOW — changes proxy routing, may break connectivity"
  - "ec2 describe-route-tables: GREEN — read-only route table query"
```

## Escalation Conditions

- DNS changes cause complete loss of connectivity to AWS services
- Proxy configuration blocks ECS/EKS control plane communication
- Network settings change causes SSM connectivity loss (locked out)
- VPC-level networking issues require network team intervention

## Data Sensitivity

- `apiclient get settings.network`: MEDIUM — reveals proxy URLs and network topology
- `apiclient get settings.dns`: LOW — returns DNS server addresses
- `aws ec2 describe-instances`: MEDIUM — reveals VPC topology and IP addresses
- Proxy configuration: MEDIUM — may contain proxy authentication details

## Prohibited Actions

- NEVER edit /etc/resolv.conf directly — use the settings API for DNS changes
- NEVER remove 169.254.169.254 from the no-proxy list — this breaks IMDS access
- NEVER change network settings without a rollback plan (risk of losing SSM access)
- NEVER configure proxy settings that block access to SSM endpoints

## Phase 3 — Rollback

1. Revert DNS settings: `apiclient set dns.name-servers=[]` (reverts to DHCP-provided DNS)
2. Remove proxy settings: `apiclient set network.https-proxy="" network.no-proxy=[]`
3. If instance is unreachable after network changes, stop and start: `aws ec2 stop-instances --instance-ids <id>` then `aws ec2 start-instances --instance-ids <id>`
4. If stop/start fails to restore connectivity, terminate and relaunch with correct user data
5. Verify connectivity is restored by checking SSM ping status

## Common Issues

- symptoms: "Containers cannot resolve DNS names but host networking works"
  diagnosis: "DNS settings in Bottlerocket are not propagated to container DNS configuration."
  resolution: "Set DNS via apiclient set dns.name-servers and dns.search-list. Reboot if needed."

- symptoms: "Instance cannot reach AWS API endpoints through proxy"
  diagnosis: "Proxy no-proxy list is missing required AWS service endpoints or CIDR ranges."
  resolution: "Add AWS service endpoints and VPC CIDR to no-proxy list via apiclient set."

## Output Format

```yaml
root_cause: "networking — <specific_cause>"
evidence:
  - type: network_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix network settings via API"
  long_term: "Configure networking in user data TOML"
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
