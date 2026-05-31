---
title: "H3 — Login and User Issues"
description: "Diagnose SQL Server login, user, and permission issues on RDS SQL Server"
status: active
severity: MEDIUM
triggers:
  - "login failed"
  - "permission denied"
  - "access denied"
  - "Error 18456"
  - "orphaned user"
  - "role membership"
  - "cannot create login"
  - "server role"
owner: devops-agent
objective: "Resolve login, user, and permission issues on RDS SQL Server"
context: "The RDS master user has processadmin, setupadmin, and db_owner on user databases — NOT sysadmin. Logins are created at the server level, users at the database level. Windows logins require domain join. The master user cannot grant sysadmin or access system databases beyond limited msdb access."
---

## Phase 1 — Triage

MUST:
- Check login exists and is enabled:
  ```sql
  SELECT name, type_desc, is_disabled, default_database_name, create_date
  FROM sys.server_principals
  WHERE type IN ('S','U','G')  -- S=SQL, U=Windows, G=Windows group
  ORDER BY name;
  ```
- Check server role membership:
  ```sql
  SELECT sp.name AS login_name, sr.name AS role_name
  FROM sys.server_role_members srm
  JOIN sys.server_principals sp ON srm.member_principal_id = sp.principal_id
  JOIN sys.server_principals sr ON srm.role_principal_id = sr.principal_id
  ORDER BY sp.name;
  ```
- Check database user mapping:
  ```sql
  USE [MyDatabase];
  SELECT dp.name AS user_name, dp.type_desc, sp.name AS login_name,
         dp.default_schema_name
  FROM sys.database_principals dp
  LEFT JOIN sys.server_principals sp ON dp.sid = sp.sid
  WHERE dp.type IN ('S','U','G') AND dp.name NOT IN ('dbo','guest','INFORMATION_SCHEMA','sys')
  ORDER BY dp.name;
  ```
- Check database role membership:
  ```sql
  USE [MyDatabase];
  SELECT dp.name AS user_name, dr.name AS role_name
  FROM sys.database_role_members drm
  JOIN sys.database_principals dp ON drm.member_principal_id = dp.principal_id
  JOIN sys.database_principals dr ON drm.role_principal_id = dr.principal_id
  ORDER BY dp.name;
  ```

SHOULD:
- Check for orphaned users:
  ```sql
  USE [MyDatabase];
  SELECT dp.name AS user_name, dp.type_desc
  FROM sys.database_principals dp
  LEFT JOIN sys.server_principals sp ON dp.sid = sp.sid
  WHERE dp.type IN ('S','U') AND sp.sid IS NULL
    AND dp.name NOT IN ('dbo','guest','INFORMATION_SCHEMA','sys');
  ```
- Check effective permissions:
  ```sql
  USE [MyDatabase];
  SELECT * FROM fn_my_permissions(NULL, 'DATABASE');
  SELECT * FROM fn_my_permissions('dbo.MyTable', 'OBJECT');
  ```
- Check for denied permissions:
  ```sql
  USE [MyDatabase];
  SELECT dp.name, dpe.permission_name, dpe.state_desc, dpe.class_desc
  FROM sys.database_permissions dpe
  JOIN sys.database_principals dp ON dpe.grantee_principal_id = dp.principal_id
  WHERE dpe.state_desc = 'DENY';
  ```

MAY:
- Check SQL Server error log for login failure details
- Verify contained database user configuration

## Phase 2 — Remediate

MUST:
- Create SQL login:
  ```sql
  CREATE LOGIN [newuser] WITH PASSWORD = 'StrongPassword123!', DEFAULT_DATABASE = [MyDatabase];
  ```
- Create database user and grant access:
  ```sql
  USE [MyDatabase];
  CREATE USER [newuser] FOR LOGIN [newuser];
  ALTER ROLE db_datareader ADD MEMBER [newuser];
  ALTER ROLE db_datawriter ADD MEMBER [newuser];
  ```
- Fix orphaned users:
  ```sql
  USE [MyDatabase];
  ALTER USER [orphaned_user] WITH LOGIN = [matching_login];
  ```

SHOULD:
- Grant specific permissions instead of broad roles:
  ```sql
  USE [MyDatabase];
  GRANT SELECT, INSERT, UPDATE ON SCHEMA::dbo TO [appuser];
  GRANT EXECUTE ON SCHEMA::dbo TO [appuser];
  ```
- For server-level roles (limited on RDS):
  ```sql
  -- Available roles for master user to grant:
  ALTER SERVER ROLE processadmin ADD MEMBER [newuser];
  ALTER SERVER ROLE setupadmin ADD MEMBER [newuser];
  -- sysadmin is NOT available
  ```

MAY:
- Implement contained database users (no server login needed):
  ```sql
  ALTER DATABASE [MyDatabase] SET CONTAINMENT = PARTIAL;
  USE [MyDatabase];
  CREATE USER [contained_user] WITH PASSWORD = 'StrongPassword123!';
  ```
- Implement application roles for fine-grained access control

## Common Issues

- symptoms: "Error 18456: Login failed for user"
  diagnosis: "Wrong password, login disabled, or login does not exist."
  resolution: "Verify login exists and is enabled. Reset password if needed."

- symptoms: "Cannot grant sysadmin role"
  diagnosis: "sysadmin is not available on RDS SQL Server."
  resolution: "Use processadmin, setupadmin, or db_owner roles instead. Use RDS-specific procedures for admin tasks."

- symptoms: "Orphaned users after database restore"
  diagnosis: "Database users have SIDs that don't match any server login."
  resolution: "Map users to logins: ALTER USER [user] WITH LOGIN = [login];"

- symptoms: "Permission denied on system objects"
  diagnosis: "Master user does not have sysadmin. Some system operations are restricted."
  resolution: "Use RDS-specific stored procedures in msdb for administrative tasks."

## Safety Ratings
- GREEN: SELECT from sys.server_principals/sys.database_principals/sys.server_role_members — read-only inspection
- YELLOW: CREATE LOGIN, ALTER LOGIN, CREATE USER, ALTER ROLE ADD MEMBER — recoverable changes
- RED: DROP LOGIN, DROP USER, delete-db-instance — destructive operations

## Escalation Conditions
- "Database serves production traffic"
- "Login/user issues affecting application connectivity"
- "Fix requires modifying server-level logins"
- "Orphaned users requiring SID remapping"
- "Data loss risk identified"

## Data Sensitivity
- HIGH: database credentials, login details, role memberships (core security infrastructure)
- HIGH: server principal details (reveal access control configuration)
- MEDIUM: database user mappings, default database settings

## Prohibited Actions
- "NEVER suggest deleting a database instance to fix login issues — use snapshots"
- "NEVER suggest disabling encryption on an encrypted database"
- "NEVER suggest granting sysadmin role"
- "NEVER suggest dropping logins without confirming they are not used by applications"

## Phase 3 — Rollback
- "Restore from snapshot if login/user changes cause access issues"
- "Re-create dropped logins if they were removed in error"
- "Revert role membership changes if they break application access"

## Output Format

```yaml
root_cause: "login_user — <specific_cause>"
evidence:
  - type: login_info
    content: "<login and role membership>"
  - type: user_mapping
    content: "<database user and permissions>"
severity: MEDIUM
mitigation:
  immediate: "Fix login, user mapping, or permissions"
  long_term: "Implement role-based access control and regular permission audits"
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
