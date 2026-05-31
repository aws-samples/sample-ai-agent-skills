---
title: "B1 — Source Endpoint Connectivity"
description: "Diagnose source endpoint connectivity failures in DMS"
status: active
severity: HIGH
triggers:
  - "source connection failed"
  - "cannot connect to source"
  - "source endpoint error"
  - "test connection failed source"
owner: devops-agent
objective: "Restore connectivity between the DMS replication instance and the source endpoint"
context: "Source endpoint connectivity failures can be caused by incorrect credentials, security group rules, network ACLs, DNS resolution, VPN/Direct Connect issues, or database listener configuration. The replication instance must be able to reach the source on the configured port."
---

## Phase 1 — Triage

MUST:
- Test the connection: `aws dms test-connection --replication-instance-arn <instance-arn> --endpoint-arn <source-endpoint-arn>`
- Check endpoint configuration: `aws dms describe-endpoints --filters Name=endpoint-id,Values=<endpoint-id> --query 'Endpoints[*].{Server:ServerName,Port:Port,Engine:EngineName,SSL:SslMode,Status:Status}'`
- Check replication instance VPC and subnets: `aws dms describe-replication-instances --filters Name=replication-instance-id,Values=<instance-id> --query 'ReplicationInstances[*].{VPC:ReplicationSubnetGroup.VpcId,Subnets:ReplicationSubnetGroup.Subnets,SGs:VpcSecurityGroups}'`
- Check security group rules allow outbound to source: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[*].IpPermissionsEgress'`

SHOULD:
- Verify DNS resolution of the source server name
- Check if source is in a different VPC (requires peering or transit gateway)
- Verify source database is listening on the configured port
- Check connection status history: `aws dms describe-connections --filters Name=endpoint-arn,Values=<endpoint-arn>`

MAY:
- Check VPC route tables for routes to the source network
- Verify NACLs allow traffic on the source port
- Test connectivity from an EC2 instance in the same subnet

## Phase 2 — Remediate

MUST:
- Fix security group rules to allow outbound traffic to source IP:port
- Correct endpoint credentials if authentication fails
- Ensure network path exists (VPC peering, VPN, Direct Connect, or public internet)

SHOULD:
- Use SSL/TLS for connections over public networks
- Configure endpoint with correct SSL mode and certificates
- Re-test connection after changes: `aws dms test-connection --replication-instance-arn <instance-arn> --endpoint-arn <endpoint-arn>`

MAY:
- Set up VPC endpoints for RDS/Aurora sources in the same region
- Configure DNS resolution for on-premises sources

## Common Issues

- symptoms: "Test connection timeout"
  diagnosis: "Network path blocked — security group, NACL, or routing issue."
  resolution: "Check security groups allow outbound on source port. Verify route tables and NACLs."

- symptoms: "Authentication failed"
  diagnosis: "Incorrect username, password, or database name in endpoint config."
  resolution: "Update endpoint credentials. Verify the user has required permissions on the source."

- symptoms: "SSL handshake failed"
  diagnosis: "SSL mode mismatch or missing CA certificate."
  resolution: "Match SSL mode to source requirements. Import CA certificate if using verify-ca."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix security group rules for outbound traffic | YELLOW | Network change — verify no unintended access |
| Correct endpoint credentials | YELLOW | Credential change — verify authentication works |
| Configure SSL/TLS for connections | GREEN | Security improvement — non-destructive |
| Set up VPC endpoints for RDS/Aurora sources | GREEN | Adds network path — non-destructive |
| Re-test connection after changes | GREEN | Verification — non-destructive |

## Escalation Conditions

- Task replicates production database
- Fix requires restarting full load
- Source connectivity loss causes growing CDC lag
- Network changes affect other services in the same VPC

## Data Sensitivity

- **Classification: HIGH**
- Endpoint configuration contains server addresses, ports, and database names
- Connection test results may reveal network topology
- Security group rules expose network access boundaries
- SSL certificate details reveal encryption configuration

## Prohibited Actions

- **NEVER** suggest deleting a replication task without confirming target data is current
- **NEVER** suggest modifying source endpoint during active CDC
- **NEVER** suggest disabling SSL/TLS to resolve connection issues
- **NEVER** suggest opening all ports in security groups to resolve connectivity

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Modified security group rules | Revert security group rules to previous configuration |
| Updated endpoint credentials | Revert to previous credentials if new ones fail |
| Configured SSL on endpoint | Revert SSL mode if it causes connection issues |

## Output Format

```yaml
root_cause: "source_connectivity — <specific_cause>"
evidence:
  - type: connection_test
    content: "<test connection result>"
  - type: endpoint_config
    content: "<endpoint configuration>"
  - type: security_groups
    content: "<security group rules>"
severity: HIGH
mitigation:
  immediate: "Fix the connectivity blocker"
  long_term: "Document network requirements and automate connectivity validation"
```

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
  - "NEVER suggest disabling SSL for replication endpoints"
  - "NEVER suggest public replication instances"
  - "NEVER suggest deleting replication tasks without data verification"
