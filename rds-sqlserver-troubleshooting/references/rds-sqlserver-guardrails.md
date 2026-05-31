# RDS SQL Server Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any RDS SQL Server issue.

## Guardrail 1: No sysadmin Role — Use Master User
The RDS master user does NOT have the sysadmin server role. The master user gets `processadmin`, `setupadmin`, and `db_owner` on user databases, with limited `msdb` access. Do not suggest granting sysadmin, running `sp_addsrvrolemember` for sysadmin, or executing commands that require sysadmin privileges.

## Guardrail 2: No OS Access
There is no RDP, SSH, or OS-level access to RDS SQL Server instances. Do not suggest accessing Windows Event Viewer, editing registry keys, modifying Windows services, or navigating the file system. Use the RDS logs API to access SQL Server error logs. `xp_cmdshell` is disabled by default and has limited functionality even when enabled.

## Guardrail 3: Multi-AZ Uses Always On AG (Not FCI)
Multi-AZ deployments on RDS SQL Server (2012+) use Always On Availability Groups under the hood, fully managed by RDS. Do not suggest manual AG configuration, `ALTER AVAILABILITY GROUP` commands, Windows Server Failover Clustering (WSFC) setup, or Failover Cluster Instances (FCI). Failover is automatic and managed by RDS.

## Guardrail 4: No SSRS, SSIS, or SSAS
SQL Server Reporting Services (SSRS), SQL Server Integration Services (SSIS), and SQL Server Analysis Services (SSAS) are NOT available on RDS SQL Server. Do not suggest installing or configuring these services. For ETL, recommend AWS Glue or SSIS on EC2. For reporting, recommend EC2-hosted SSRS or Amazon QuickSight.

## Guardrail 5: Native Backup/Restore via msdb Stored Procedures Only
Native backup and restore on RDS SQL Server uses the `msdb.dbo.rds_backup_database` and `msdb.dbo.rds_restore_database` stored procedures to interact with S3. Do not suggest using `BACKUP DATABASE ... TO DISK` or `RESTORE DATABASE ... FROM DISK` directly. S3 integration requires an option group with `SQLSERVER_BACKUP_RESTORE` and an associated IAM role.

## Guardrail 6: Collation Is Immutable After Creation
The server-level collation is set at instance creation and cannot be changed afterward. Default is `SQL_Latin1_General_CP1_CI_AS`. Do not suggest changing server collation or reinstalling SQL Server. Database-level collation can be set independently at database creation but server collation is permanent. If server collation must change, create a new instance and migrate data.

## Guardrail 7: TDE Requires Enterprise Edition
Transparent Data Encryption (TDE) on RDS SQL Server requires Enterprise Edition. TDE is NOT available on Express, Web, or Standard editions. TDE is configured via option group and uses AWS KMS for key management. Do not suggest enabling TDE on non-Enterprise editions.

## Guardrail 8: SQL Server Agent Limited — No PowerShell, No OS Commands
SQL Server Agent is available on Web, Standard, and Enterprise editions (not Express) but is limited. PowerShell job steps, CmdExec (OS command) steps, and ActiveX steps are NOT supported. Only T-SQL and SSIS package execution steps are available. Multi-server administration (MSX/TSX) is not supported.

## Guardrail 9: CLR Limited — SAFE Assemblies Only by Default
CLR integration supports only SAFE assemblies by default. EXTERNAL_ACCESS and UNSAFE assemblies require additional configuration and are restricted on RDS. Do not suggest deploying UNSAFE CLR assemblies without verifying RDS support for the specific scenario.

## Guardrail 10: Tempdb Managed by RDS
RDS manages tempdb file count and sizing automatically. On SQL Server 2016+, RDS creates multiple tempdb data files based on vCPU count (up to 8 files). You cannot add or remove tempdb files manually. Do not suggest `ALTER DATABASE tempdb ADD FILE` or manual tempdb file management.

## Guardrail 11: Max Server Memory Auto-Managed
RDS automatically calculates and sets `max server memory` based on the instance class. Do not override this value via parameter group unless specifically troubleshooting memory issues, as incorrect values can cause instance instability or OS-level memory pressure.

## Guardrail 12: No Linked Servers Without Option Group
Linked servers are not available by default on RDS SQL Server. They require option group configuration. Even with option groups, linked servers are limited to SQL Server targets. OLE DB providers for Oracle, MySQL, or other databases are not available on RDS. Do not suggest creating linked servers to non-SQL Server targets.
