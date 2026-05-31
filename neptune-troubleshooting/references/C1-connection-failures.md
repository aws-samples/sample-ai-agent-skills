---
title: "C1 — Connection Failures"
description: "Diagnose Neptune connection failures and timeouts"
status: active
severity: HIGH
triggers:
  - "cannot connect"
  - "connection refused"
  - "connection timeout"
  - "endpoint unreachable"
owner: devops-agent
objective: "Identify and resolve Neptune connectivity issues"
context: "Neptune is VPC-only. Connection failures are typically caused by security group rules, subnet configuration, DNS resolution, or network ACLs. There is no public endpoint option."
---

## Phase 1 — Triage

MUST:
- Verify cluster is available: `aws neptune describe-db-clusters --db-cluster-identifier <cluster-id> --query 'DBClusters[0].Status'`
- Check security group allows port 8182: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Verify client is in same VPC or has connectivity: `aws ec2 describe-instances --instance-ids <client-instance-id> --query 'Reservations[0].Instances[0].{VpcId:VpcId,SubnetId:SubnetId}'`
- Test connectivity: `curl -s -o /dev/null -w "%{http_code}" https://<cluster-endpoint>:8182/status`

SHOULD:
- Check subnet routing: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`
- Verify DNS resolution: `nslookup <cluster-endpoint>`
- Check network ACLs: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>`
- Verify cluster endpoints: `aws neptune describe-db-cluster-endpoints --db-cluster-identifier <cluster-id>`

MAY:
- Check VPC peering if cross-VPC: `aws ec2 describe-vpc-peering-connections`
- Verify Transit Gateway routes if applicable
- Test from different subnets to isolate network issue

## Phase 2 — Remediate

MUST:
- Fix security group to allow inbound TCP 8182 from client CIDR/security group
- Ensure client and Neptune are in the same VPC or have network path
- Verify DNS resolution works for the cluster endpoint

SHOULD:
- Use cluster endpoint for writes, reader endpoint for reads
- Configure connection pooling in application
- Set appropriate connection timeouts

MAY:
- Set up VPC endpoint for Neptune if needed
- Configure bastion host for external access

## Common Issues

- symptoms: "Connection timeout from EC2 instance"
  diagnosis: "Security group does not allow inbound TCP 8182 from the client."
  resolution: "Add inbound rule for TCP 8182 from the client security group or CIDR."

- symptoms: "Connection refused from outside VPC"
  diagnosis: "Neptune is VPC-only. No public endpoint exists."
  resolution: "Use bastion host, VPN, or ALB proxy for external access."

- symptoms: "Intermittent connection failures"
  diagnosis: "DNS caching returning stale endpoint after failover."
  resolution: "Set DNS TTL to 5 seconds or less. Use cluster endpoint for automatic failover."

## Output Format

```yaml
root_cause: "connection_failure — <specific_cause>"
evidence:
  - type: cluster_status
    content: "<cluster availability status>"
  - type: security_group
    content: "<security group rules>"
  - type: network
    content: "<connectivity test results>"
severity: HIGH
mitigation:
  immediate: "Fix network connectivity"
  long_term: "Document network architecture and access patterns"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: LOW — connection failures prevent data access
- Service Disruption Risk: HIGH — connection failures directly impact application availability
- Compliance Impact: LOW — connection issues are operational concerns

## Escalation Conditions
- All connections to the cluster fail simultaneously
- Connection failures persist after security group and network verification
- SSL/TLS handshake failures after certificate rotation
- Connection pool exhaustion causing application-wide outages
- DNS resolution failures for Neptune endpoints

## Data Sensitivity
- Connection strings and endpoints — INTERNAL
- SSL/TLS certificates — INTERNAL
- Security group configurations — INTERNAL
- VPC and subnet details — INTERNAL
- Never expose database connection credentials

## Prohibited Actions
- Do NOT disable SSL/TLS for Neptune connections
- Do NOT open Neptune security groups to 0.0.0.0/0
- Do NOT bypass IAM authentication to troubleshoot connection issues
- Do NOT expose Neptune endpoints outside the VPC without proper controls
- Do NOT ignore connection timeout patterns
- Do NOT modify Neptune port settings without updating all clients

## Phase 3 — Rollback
- Restore previous security group rules if changes caused connection failures
- Revert DNS configuration changes
- Restore SSL/TLS certificate configuration
- Re-enable IAM authentication if temporarily disabled
- Verify all application connections are restored after changes
- Document connection failure timeline and resolution steps

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
  - "NEVER suggest disabling IAM authentication"
  - "NEVER suggest public access"
  - "NEVER suggest disabling encryption at rest"
