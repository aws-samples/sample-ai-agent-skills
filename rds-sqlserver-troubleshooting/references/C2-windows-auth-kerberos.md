---
title: "C2 — Windows Authentication / Kerberos"
description: "Diagnose Windows Authentication and Kerberos issues on RDS SQL Server"
status: active
severity: MEDIUM
triggers:
  - "Windows Authentication"
  - "Kerberos"
  - "Active Directory"
  - "domain join"
  - "NTLM"
  - "SPN"
  - "trust relationship"
owner: devops-agent
objective: "Resolve Windows Authentication and Kerberos configuration issues on RDS SQL Server"
context: "Windows Authentication on RDS SQL Server requires AWS Managed Microsoft AD (via AWS Directory Service). The RDS instance must be domain-joined via an option group. Kerberos is used for authentication. Self-managed AD can be connected via AD trust or AD Connector. NTLM is not supported — only Kerberos."
---

## Phase 1 — Triage

MUST:
- Check if instance is domain-joined: `aws rds describe-db-instances --db-instance-identifier <instance-id> --query 'DBInstances[0].DomainMemberships'`
- Check Directory Service status: `aws ds describe-directories`
- Check option group for Active Directory: `aws rds describe-option-groups --option-group-name <option-group> --query "OptionGroupsList[0].Options[?OptionName=='SQLSERVER_AUDIT']"`
- Check RDS events for domain join issues: `aws rds describe-events --source-identifier <instance-id> --source-type db-instance --duration 1440`

SHOULD:
- Verify the AWS Managed Microsoft AD is in the same VPC or has VPC peering/connectivity
- Check DNS resolution from the RDS VPC to the AD domain
- Verify Windows logins exist on the SQL Server:
  ```sql
  SELECT name, type_desc, default_database_name, is_disabled
  FROM sys.server_principals
  WHERE type IN ('U','G')  -- U=Windows login, G=Windows group
  ORDER BY name;
  ```
- Check for Kerberos-related errors in SQL Server error log:
  ```
  aws rds download-db-log-file-portion --db-instance-identifier <instance-id> --log-file-name error/ERRORLOG
  ```

MAY:
- Verify AD trust relationship (if using self-managed AD with trust)
- Check SPN registration for the RDS instance
- Verify client-side Kerberos ticket: `klist` on the client machine

## Phase 2 — Remediate

MUST:
- For domain join: configure via modify-db-instance with directory ID
  ```
  aws rds modify-db-instance \
    --db-instance-identifier <instance-id> \
    --domain <directory-id> \
    --domain-iam-role-name <iam-role> \
    --apply-immediately
  ```
- For missing Windows logins: create them via T-SQL
  ```sql
  CREATE LOGIN [DOMAIN\username] FROM WINDOWS;
  -- Or for a group
  CREATE LOGIN [DOMAIN\groupname] FROM WINDOWS;
  ```
- For domain join failures: verify IAM role has required permissions for Directory Service

SHOULD:
- Ensure the IAM role for domain join has the `AmazonRDSDirectoryServiceAccess` managed policy
- Verify VPC security groups allow AD traffic (ports 53, 88, 135, 389, 445, 464, 636, 3268, 3269, 49152-65535)
- Grant database access to Windows logins:
  ```sql
  USE [MyDatabase];
  CREATE USER [DOMAIN\username] FOR LOGIN [DOMAIN\username];
  ALTER ROLE db_datareader ADD MEMBER [DOMAIN\username];
  ```

MAY:
- Configure AD trust for self-managed AD: `aws ds create-trust --directory-id <dir-id> --remote-domain-name <domain> --trust-direction Two-Way --trust-password <password>`
- Test Windows Auth from a domain-joined EC2 instance

## Common Issues

- symptoms: "Login failed — Windows Authentication not working"
  diagnosis: "Instance not domain-joined or domain join failed."
  resolution: "Verify DomainMemberships status. Re-join domain if needed."

- symptoms: "Domain join status shows 'failed'"
  diagnosis: "IAM role missing permissions, VPC DNS not configured, or AD unreachable."
  resolution: "Check IAM role. Verify VPC DNS settings point to AD DNS. Check security groups for AD ports."

- symptoms: "Kerberos authentication fails, falls back to error"
  diagnosis: "SPN not registered or DNS resolution failure. NTLM fallback is not supported on RDS."
  resolution: "Verify DNS resolution. Check AD health. Ensure client can obtain Kerberos ticket."

## Safety Ratings
- GREEN: describe-db-instances (DomainMemberships), ds describe-directories, SELECT from sys.server_principals, error log — read-only inspection
- YELLOW: modify-db-instance (domain join), CREATE LOGIN FROM WINDOWS — recoverable changes
- RED: delete-db-instance, force-failover — destructive or high-impact operations

## Escalation Conditions
- "Database serves production traffic"
- "Windows Authentication failure affecting all domain users"
- "Fix requires re-joining domain (causes brief downtime)"
- "AD trust relationship broken"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: Active Directory credentials, domain configuration, Kerberos tickets
- HIGH: Windows login details, domain membership configuration
- MEDIUM: AD directory status, SPN registration, error log entries

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix auth issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest granting sysadmin role"
- "NEVER suggest storing AD credentials in application code"

## Phase 3 — Rollback
- "If domain join fails, revert to SQL Authentication temporarily"
- "Revert domain membership changes if they break authentication"
- "If failover causes issues, wait for DNS propagation (do NOT failover again immediately)"

## Output Format

```yaml
root_cause: "windows_auth — <specific_cause>"
evidence:
  - type: domain_membership
    content: "<DomainMemberships status>"
  - type: directory_service
    content: "<AD directory status>"
  - type: error_log
    content: "<relevant error log entries>"
severity: MEDIUM
mitigation:
  immediate: "Fix domain join or Windows login configuration"
  long_term: "Implement AD monitoring and automate Windows login provisioning"
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
  - "NEVER suggest making databases publicly accessible"
  - "NEVER suggest disabling encryption at rest"
  - "NEVER suggest deleting automated backups"
