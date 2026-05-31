# RDS Oracle Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any RDS Oracle issue.

## Guardrail 1: No RAC on RDS
Oracle Real Application Clusters (RAC) is not available on Amazon RDS. Do not suggest RAC configuration, RAC-specific wait events troubleshooting, or cluster interconnect diagnostics. For high availability, use Multi-AZ which uses Data Guard internally.

## Guardrail 2: No Direct RMAN Access
RMAN (Recovery Manager) is not directly accessible on RDS Oracle. Do not suggest running `rman target /` or RMAN commands. Automated backups use storage-level snapshots. For logical backups, use Data Pump via the `rdsadmin.rdsadmin_datapump_util` procedures.

## Guardrail 3: Multi-AZ Uses Data Guard (Not Customer-Managed)
Multi-AZ deployments use Oracle Data Guard (physical standby) under the hood, but it is fully managed by RDS. Do not suggest manual Data Guard commands like `ALTER DATABASE SWITCHOVER`, `DGMGRL`, or standby redo log configuration. Failover is automatic and managed by RDS.

## Guardrail 4: Read Replicas Require EE with Active Data Guard
Read replicas for RDS Oracle require Enterprise Edition with an Active Data Guard license. They are not available on Standard Edition Two (SE2). Do not suggest creating read replicas on SE2 instances.

## Guardrail 5: Character Set Is Immutable After Creation
The database character set (default AL32UTF8) and national character set (AL16UTF16) cannot be changed after instance creation. Do not suggest `ALTER DATABASE CHARACTER SET` or `CSSCAN`/`CSALTER` utilities. If a character set change is needed, the data must be migrated to a new instance with the correct character set.

## Guardrail 6: BYOL Licensing Requirements
Bring Your Own License (BYOL) requires existing Oracle licenses with active Software Update License & Support (SULS). License Included is available for SE2 only. Enterprise Edition always requires BYOL. Verify licensing compliance before recommending edition changes.

## Guardrail 7: Oracle-Specific Parameter Constraints
Not all Oracle init.ora parameters are modifiable on RDS. Parameters like `db_block_size`, `compatible`, and `control_files` are managed by RDS and cannot be changed. Always check `describe-db-parameters` with `--source user` to see which parameters are modifiable. Some parameter changes require a reboot (static parameters).

## Guardrail 8: No OS Access — Use Option Groups for Features
There is no SSH or OS-level access to RDS Oracle instances. Features that normally require OS access (TDE, SSL, Spatial, XMLDB, Application Express) are enabled via RDS option groups. Do not suggest editing `sqlnet.ora`, `listener.ora`, or `tnsnames.ora` files directly.

## Guardrail 9: TDE with KMS vs CloudHSM
Transparent Data Encryption (TDE) on RDS Oracle requires Enterprise Edition with the Advanced Security option. TDE can use AWS KMS (default, simpler) or AWS CloudHSM (for regulatory requirements needing single-tenant HSM). TDE is configured via option groups, not SQL commands. Do not suggest `ALTER SYSTEM SET ENCRYPTION KEY` directly.

## Guardrail 10: Oracle Version EOL and Upgrade Paths
Not all Oracle versions can upgrade directly to the latest version. Use `aws rds describe-db-engine-versions --engine oracle-ee --engine-version <current>` to check valid upgrade targets. Major version upgrades may require parameter group family changes and application testing. RDS deprecates Oracle versions on a schedule — plan upgrades before forced deprecation.

## Guardrail 11: SGA/PGA Auto-Management in RDS
RDS manages Oracle memory allocation based on the instance class. The `DBInstanceClassMemory` formula variable is used in parameter group defaults. Do not set `sga_target` or `pga_aggregate_target` to values exceeding available instance memory. Over-allocating causes ORA-04031 (shared pool) or instance instability.

## Guardrail 12: No Direct Access to Alert Log — Use RDS Events and Logs API
The Oracle alert log is not accessible via the filesystem. Use `aws rds download-db-log-file-portion --log-file-name trace/alert_<SID>.log` to retrieve it. Listener logs, trace files, and audit logs are also accessed via the RDS logs API. Do not suggest navigating to `$ORACLE_HOME/diag/` or `$ORACLE_BASE/admin/`.
