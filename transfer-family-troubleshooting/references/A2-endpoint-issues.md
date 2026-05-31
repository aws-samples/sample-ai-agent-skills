---
title: "A2 — Transfer Family Endpoint Configuration Issues"
description: "Diagnose endpoint type misconfigurations including PUBLIC, VPC, and VPC_ENDPOINT"
status: active
severity: HIGH
triggers:
  - "endpoint not reachable"
  - "VPC endpoint issue"
  - "Elastic IP not working"
  - "endpoint type"
  - "static IP"
  - "PUBLIC endpoint"
owner: devops-agent
objective: "Identify and resolve endpoint configuration issues affecting server accessibility"
context: "Transfer Family supports three endpoint types: PUBLIC (AWS-managed, no security groups), VPC (ENIs in your VPC with optional EIPs and security groups), and VPC_ENDPOINT (legacy PrivateLink). Each has different networking characteristics and capabilities."
---

## Phase 1 — Triage

MUST:
- Check endpoint type and details: `aws transfer describe-server --server-id <server-id> --query 'Server.{EndpointType:EndpointType,EndpointDetails:EndpointDetails}'`
- For VPC type, check ENIs: `aws ec2 describe-network-interfaces --filters Name=description,Values="*<server-id>*"`
- For VPC type, check security groups on ENIs: `aws ec2 describe-security-groups --group-ids <sg-ids-from-enis>`
- Check if Elastic IPs are associated (VPC type): `aws ec2 describe-addresses --filters Name=network-interface-id,Values=<eni-id>`
- Verify DNS resolution of the server hostname: `nslookup <server-id>.server.transfer.<region>.amazonaws.com`

SHOULD:
- For VPC_ENDPOINT type, check VPC endpoint status: `aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <vpce-id>`
- Check subnet route tables for internet access (if public-facing VPC endpoint): `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`
- Verify NACLs on subnets allow required ports: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>`

MAY:
- Check if a custom hostname is configured with Route 53
- Verify NLB configuration if using NLB in front of VPC endpoint

## Phase 2 — Remediate

MUST:
- For static public IPs: use VPC endpoint type and associate Elastic IPs to the ENIs
- Ensure security groups (VPC type) allow inbound on required protocol ports from client CIDRs
- Ensure NACLs allow both inbound and outbound traffic on required ports (including ephemeral ports)

SHOULD:
- Migrate from VPC_ENDPOINT (legacy) to VPC endpoint type for new deployments
- Use multiple subnets in different AZs for high availability
- Document the endpoint architecture for operational reference

MAY:
- Configure a custom hostname via Route 53 CNAME pointing to the server endpoint
- Set up NLB for additional load balancing or fixed IP requirements

## Common Issues

- symptoms: "Cannot connect to PUBLIC endpoint from corporate network"
  diagnosis: "Corporate firewall blocks outbound SSH (port 22) or FTP ports."
  resolution: "Work with network team to allow outbound traffic on required ports. Consider VPC endpoint with VPN."

- symptoms: "VPC endpoint has no public access despite Elastic IPs"
  diagnosis: "Elastic IPs not associated with the VPC endpoint ENIs, or subnet route table lacks internet gateway route."
  resolution: "Associate EIPs to ENIs. Ensure subnet route table has 0.0.0.0/0 → igw route."

- symptoms: "VPC_ENDPOINT only accessible from within VPC"
  diagnosis: "VPC_ENDPOINT type uses PrivateLink — by design only accessible from within the VPC or peered/connected networks."
  resolution: "Migrate to VPC endpoint type for public access with static IPs, or use VPN/Direct Connect for private access."

## Output Format

```yaml
root_cause: "endpoint_issue — <specific_cause>"
evidence:
  - type: endpoint_config
    content: "<endpoint type and details>"
  - type: network
    content: "<security groups, NACLs, route tables>"
  - type: dns
    content: "<DNS resolution results>"
severity: HIGH
mitigation:
  immediate: "Fix endpoint configuration or network rules"
  long_term: "Document endpoint architecture and implement monitoring"
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

- Transferred files may contain any type of sensitive data
- SSH keys and credentials enable server access
- Directory mappings reveal S3/EFS data organization
- Classification: CONFIDENTIAL - file transfer content and access credentials

## Prohibited Actions

- NEVER modify production server endpoints during active file transfers
- NEVER change authentication providers without user notification
- NEVER delete SSH host keys without coordinating with all SFTP clients
- NEVER modify directory mappings that could expose unauthorized S3/EFS paths

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
  - "NEVER suggest disabling SFTP encryption"
  - "NEVER suggest FTP protocol for production"
  - "NEVER suggest overly broad IAM roles for transfer users"
