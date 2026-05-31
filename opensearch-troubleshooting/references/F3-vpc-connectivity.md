---
title: "F3 — OpenSearch VPC Connectivity"
description: "Diagnose and resolve VPC domain connectivity issues"
status: active
severity: HIGH
triggers:
  - "VPC connectivity"
  - "cannot connect"
  - "connection refused"
  - "timeout"
  - "VPC domain"
  - "security group"
  - "subnet"
owner: devops-agent
objective: "Resolve VPC connectivity issues and establish access to the OpenSearch domain"
context: "OpenSearch domains deployed in a VPC have no public endpoint. Access requires being in the VPC or using VPN, VPC peering, Transit Gateway, or a reverse proxy. Security groups control inbound access (port 443 for HTTPS). The domain is placed in the specified subnets with ENIs managed by the service. VPC vs public access cannot be changed after domain creation."
---

## Phase 1 — Triage

MUST:
- Check VPC configuration: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.VPCOptions'`
- Check security groups: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.VPCOptions.SecurityGroupIds'`
- Check security group rules: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[*].IpPermissions'`
- Check subnets: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.VPCOptions.SubnetIds'`
- Check domain endpoint: `aws opensearch describe-domain --domain-name <domain> --query 'DomainStatus.Endpoints'`

SHOULD:
- Check subnet route tables: `aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=<subnet-id>"`
- Check network ACLs: `aws ec2 describe-network-acls --filters "Name=association.subnet-id,Values=<subnet-id>"`
- Verify client is in the same VPC or has connectivity (VPN, peering, TGW)
- Check ENIs created by OpenSearch: `aws ec2 describe-network-interfaces --filters "Name=requester-id,Values=*opensearch*"`

MAY:
- Test connectivity from within VPC: `curl -XGET "https://<endpoint>/_cluster/health" --connect-timeout 5`
- Check DNS resolution: `nslookup <endpoint>`
- Verify VPC peering or Transit Gateway routes if connecting from another VPC

## Phase 2 — Remediate

MUST:
- If security group blocks access: add inbound rule for port 443 from client CIDR: `aws ec2 authorize-security-group-ingress --group-id <sg-id> --protocol tcp --port 443 --cidr <client-cidr>`
- If client not in VPC: set up VPN, VPC peering, or reverse proxy
- If DNS not resolving: check VPC DNS settings (enableDnsHostnames and enableDnsSupport must be true)

SHOULD:
- Use a bastion host or SSH tunnel for ad-hoc access: `ssh -L 9200:<endpoint>:443 ec2-user@<bastion>`
- Set up Nginx reverse proxy on EC2 for Dashboards access from outside VPC
- Ensure subnets have routes to the client network

MAY:
- Use AWS Client VPN for developer access to VPC domains
- Set up Transit Gateway for multi-VPC access
- Consider creating a public domain if VPC access is too complex (requires new domain)

## Common Issues

- symptoms: "Connection timeout from local machine"
  diagnosis: "VPC domain has no public endpoint. Cannot connect from internet."
  resolution: "Use VPN, bastion host, or reverse proxy. VPC domains are not publicly accessible."

- symptoms: "Connection refused from EC2 in same VPC"
  diagnosis: "Security group not allowing inbound on port 443 from EC2's security group or CIDR."
  resolution: "Add inbound rule to OpenSearch security group for port 443 from client."

- symptoms: "DNS resolution fails for domain endpoint"
  diagnosis: "VPC DNS settings not enabled or client not using VPC DNS."
  resolution: "Enable enableDnsHostnames and enableDnsSupport on the VPC."

## Output Format

```yaml
root_cause: "vpc_connectivity — <specific_cause>"
evidence:
  - type: vpc_config
    content: "<VPC, subnets, security groups>"
  - type: security_group_rules
    content: "<inbound rules for port 443>"
  - type: connectivity_test
    content: "<connection test result>"
severity: HIGH
mitigation:
  immediate: "Fix security group rules or establish VPC connectivity path"
  long_term: "Document VPC access architecture, set up persistent connectivity"
```


## Safety Ratings
```
safety_ratings:
  - "Check VPC configuration and security groups: GREEN — read-only API calls"
  - "Check subnet route tables: GREEN — read-only inspection"
  - "Add security group inbound rule: YELLOW — changes network access"
  - "Set up VPN or bastion host: YELLOW — creates new access path"
  - "Set up reverse proxy: YELLOW — creates new access path with security implications"
```

## Escalation Conditions
- Domain serves production search
- VPC connectivity issues blocking production applications
- Security group changes affecting other services
- Cross-VPC access requiring networking team coordination
- DNS resolution failures suggesting VPC-level issues

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "VPC configuration: network topology"
    - "Security group rules: access control configuration"
    - "Domain endpoint: connection details"
    - "Subnet IDs: infrastructure layout"
  handling: "Do not expose VPC topology, security group rules, or domain endpoints externally."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER open port 443 to 0.0.0.0/0 on the domain security group
- NEVER change VPC/public access mode (cannot be changed after domain creation)

## Phase 3 — Rollback
- If security group rules were added: remove the added rules
- If bastion host was set up: terminate the instance if not needed
- If reverse proxy was configured: remove the proxy configuration
- If VPC peering was created: delete peering and remove route table entries

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling fine-grained access control"
  - "NEVER suggest public access domains"
  - "NEVER suggest disabling encryption at rest"
