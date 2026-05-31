---
title: "C2 — Database Connectivity Issues"
description: "Diagnose connectivity problems between Lightsail instances and managed databases"
status: active
severity: HIGH
triggers:
  - "cannot connect to database"
  - "database connection refused"
  - "database timeout"
  - "authentication failed"
owner: devops-agent
objective: "Identify and resolve database connectivity issues"
context: "Lightsail managed databases can be accessed from Lightsail instances in the same region. Public mode allows external access. Database endpoint, port, credentials, and network configuration must all be correct. Default ports: MySQL 3306, PostgreSQL 5432."
---

## Phase 1 — Triage

MUST:
- Get database endpoint: `aws lightsail get-relational-database --relational-database-name <db-name> --query 'relationalDatabase.masterEndpoint'`
- Check if public mode is enabled: `aws lightsail get-relational-database --relational-database-name <db-name> --query 'relationalDatabase.publiclyAccessible'`
- Verify database is in available state
- Test connectivity from instance to database endpoint on correct port

SHOULD:
- Verify master username and password
- Check if connecting from Lightsail instance (same region) or external
- Verify database engine version compatibility with client

MAY:
- Check database parameter group for connection settings
- Review connection limits and current connections

## Phase 2 — Remediate

MUST:
- Use correct endpoint, port, username, and password
- For external access, enable public mode: `aws lightsail update-relational-database --relational-database-name <db-name> --publicly-accessible`
- Ensure database is in available state

SHOULD:
- Connect from Lightsail instance in same region for best connectivity
- Use SSL/TLS for database connections

MAY:
- Reset master password if forgotten: `aws lightsail update-relational-database --relational-database-name <db-name> --master-user-password <new-password>`
- Configure connection pooling for high-connection workloads

## Common Issues

- symptoms: "Connection refused from external client"
  diagnosis: "Database public mode not enabled."
  resolution: "Enable public mode or connect from a Lightsail instance in the same region."

- symptoms: "Authentication failed"
  diagnosis: "Wrong username or password."
  resolution: "Verify credentials. Reset password if needed."

- symptoms: "Connection timeout"
  diagnosis: "Database not available or network path blocked."
  resolution: "Check database state. Verify network connectivity."

## Output Format

```yaml
root_cause: "database_connectivity — <specific_cause>"
evidence:
  - type: endpoint
    content: "<database endpoint and port>"
  - type: public_access
    content: "<publicly accessible setting>"
  - type: connectivity_test
    content: "<connection test result>"
severity: HIGH
mitigation:
  immediate: "Fix connectivity configuration"
  long_term: "Document database access requirements"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** delete a static IP address attached to a production instance without reassignment planning
- **NEVER** modify firewall rules to allow unrestricted inbound access (0.0.0.0/0) on sensitive ports
- **NEVER** delete snapshots that serve as the only backup for production instances or databases
- **NEVER** force-stop a managed database instance without verifying active connections and transactions
- **NEVER** detach a disk from a running instance without first unmounting it from the operating system

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
