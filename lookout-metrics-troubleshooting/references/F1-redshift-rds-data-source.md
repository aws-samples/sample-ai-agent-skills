---
title: "F1 — Redshift/RDS Data Source"
description: "Diagnose and resolve Redshift and RDS data source connectivity and query issues"
status: active
severity: HIGH
triggers: ["Redshift data source error", "RDS connectivity failed", "database query error", "VPC connectivity issue"]
owner: devops-agent
objective: "Ensure reliable data ingestion from Redshift and RDS data sources into Lookout for Metrics"
context: "Redshift and RDS data sources require VPC connectivity, proper security group configuration, database credentials, and correct SQL query configuration. Network connectivity is the most common failure point."
---

## Phase 1 — Triage

MUST:
- Describe metric set to get database configuration: `aws lookoutmetrics describe-metric-set --metric-set-arn <arn>`
- For Redshift: verify cluster is available: `aws redshift describe-clusters --cluster-identifier <id> --query 'Clusters[0].{Status:ClusterStatus,Endpoint:Endpoint}'`
- For RDS: verify instance is available: `aws rds describe-db-instances --db-instance-identifier <id> --query 'DBInstances[0].{Status:DBInstanceStatus,Endpoint:Endpoint}'`

SHOULD:
- Check security group rules allow inbound from Lookout for Metrics: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Verify VPC subnet routing and NAT gateway configuration
- Check database credentials are valid and not expired

MAY:
- Test database connectivity from a bastion host in the same VPC
- Review VPC flow logs for connection attempts from Lookout for Metrics

## Phase 2 — Remediate

MUST:
- Fix security group rules to allow inbound on the database port from Lookout for Metrics
- Verify database credentials in Secrets Manager are current
- Ensure the VPC subnet has proper routing (NAT gateway for private subnets)

SHOULD:
- Verify the SQL query returns data in the expected format with correct column names
- Check database user has SELECT permissions on the required tables
- Ensure Redshift cluster or RDS instance is not in maintenance window

MAY:
- Set up VPC endpoints to avoid NAT gateway for Lookout for Metrics traffic
- Implement database connection pooling for high-frequency detectors

## Safety Ratings

```
safety_ratings:
  - "redshift describe-clusters: GREEN — read-only cluster status"
  - "rds describe-db-instances: GREEN — read-only instance status"
  - "ec2 describe-security-groups: GREEN — read-only security group check"
  - "Security group rule modification: YELLOW — changes network access"
```

## Escalation Conditions

- Database is in a private subnet with no NAT gateway and VPC endpoints are not available
- Security group changes require network team approval
- Database credentials are managed by a different team and cannot be updated
- Redshift cluster or RDS instance is at capacity and cannot handle additional query load

## Data Sensitivity

- `aws redshift describe-clusters`: MEDIUM — reveals cluster endpoint and configuration
- `aws rds describe-db-instances`: MEDIUM — reveals instance endpoint and configuration
- Database credentials: CRITICAL — must be stored in Secrets Manager, never in plaintext
- SQL query results: HIGH — contain actual business data
- VPC configuration: MEDIUM — reveals network architecture

## Prohibited Actions

- NEVER store database credentials in plaintext in metric set configuration
- NEVER open security groups to 0.0.0.0/0 for database connectivity
- NEVER run destructive SQL queries (DELETE, DROP, TRUNCATE) during troubleshooting
- NEVER expose database endpoints or credentials in logs or outputs
- NEVER modify database schema to fix Lookout for Metrics query issues without DBA approval

## Phase 3 — Rollback

1. If security group rules were added for testing, remove them after resolution
2. If database credentials were rotated, update Secrets Manager and metric set configuration
3. If VPC routing was changed, revert to previous configuration
4. Document all network and database configuration changes
5. Verify data ingestion resumes by checking data quality metrics after fixes

## Common Issues

- symptoms: "Database data source returns connection timeout"
  diagnosis: "Security group does not allow inbound on the database port, or VPC routing is incorrect."
  resolution: "Add security group rule for the database port. Verify subnet routing and NAT gateway."

- symptoms: "Database query returns no results"
  diagnosis: "SQL query column names do not match metric set configuration, or query filter excludes all data."
  resolution: "Test the SQL query directly against the database. Verify column names match metric set measures and dimensions."

## Output Format

```yaml
root_cause: "database_data_source — <specific_cause>"
evidence:
  - type: database_connectivity
    content: "<database status, security group rules, and query details>"
severity: HIGH
mitigation:
  immediate: "Fix network connectivity or query configuration"
  long_term: "Implement database connectivity monitoring and credential rotation"
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
