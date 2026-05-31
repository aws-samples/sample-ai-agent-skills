# AWS DMS Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS DMS issue.

## Guardrail 1: CDC Requires Source-Specific Supplemental Logging
CDC does not work out of the box. Oracle requires supplemental logging enabled at database or table level. MySQL requires binlog_format=ROW and binlog_row_image=FULL. PostgreSQL requires wal_level=logical and a logical replication slot. SQL Server requires MS-CDC or MS-Replication enabled. Without these, DMS cannot capture ongoing changes.

## Guardrail 2: LOB Handling Mode Dramatically Affects Performance
Full LOB mode reads each LOB value individually from the source — extremely slow for large tables. Limited LOB mode truncates LOBs at the configured max size — fast but may lose data. Inline LOB mode handles small LOBs inline and large LOBs separately — best balance. Always check LOB column sizes before choosing a mode. Default is limited LOB mode.

## Guardrail 3: Replication Instance Class Determines Throughput Ceiling
The replication instance class (dms.t3.medium, dms.r5.large, etc.) sets CPU, memory, and network limits. Undersized instances cause task slowness, not errors. Monitor CPUUtilization, FreeableMemory, and NetworkTransmitThroughput. Scale up the instance class for large or high-throughput migrations. T-class instances have burstable CPU — avoid for sustained workloads.

## Guardrail 4: Replication Instance Storage Can Fill During Migration
DMS uses local storage for change caching, sorting, and transaction logs. Large transactions, LOB data, and high CDC volume consume storage rapidly. When storage is full, the task stops. Monitor FreeStorageSpace metric. Increase allocated storage proactively. Storage cannot be decreased after allocation.

## Guardrail 5: Table Mapping Rules Are Evaluated in Order
Selection rules and transformation rules in table mappings are evaluated top to bottom. The first matching rule wins for selection. Wildcards (%) match any string. Incorrect rule ordering can include or exclude wrong tables. Always test table mappings with a dry run before full migration. Use describe-table-statistics to verify which tables are included.

## Guardrail 6: Data Validation During CDC May Show Transient Mismatches
Data validation compares source and target rows. During active CDC, the source may change between validation reads, causing false mismatches. Validation adds overhead to the migration. Re-run validation after CDC catches up for accurate results. Validation does not compare LOB columns by default.

## Guardrail 7: Multi-AZ Provides HA, Not Performance
Multi-AZ replication instances have a standby in another AZ for automatic failover. The standby does NOT process migration tasks. Multi-AZ doubles the cost. It does not improve throughput or reduce latency. Use Multi-AZ for production migrations where downtime must be minimized.

## Guardrail 8: VPC Configuration Must Allow Source and Target Connectivity
The replication instance must reach both source and target endpoints. For RDS/Aurora in the same VPC, security groups must allow inbound from the replication instance. For on-premises sources, VPN or Direct Connect is required. Public endpoints need the replication instance in a public subnet with a public IP. Check security groups, NACLs, and route tables.

## Guardrail 9: Task Settings JSON Controls Detailed Migration Behavior
Task settings (JSON) control parallel load threads, batch apply, error handling, logging, and more. Default settings are conservative. BatchApplyEnabled=true improves CDC apply performance. ParallelLoadThreads increases full load speed. MaxFullLoadSubTasks controls concurrent table loads. Incorrect settings can cause data integrity issues.

## Guardrail 10: Schema Conversion and Data Migration Are Separate Steps
AWS Schema Conversion Tool (SCT) converts database schema (DDL). DMS migrates data (DML). For heterogeneous migrations (e.g., Oracle to PostgreSQL), run SCT first to convert schema, then DMS to migrate data. DMS can create basic target tables but does not convert stored procedures, triggers, or views.

## Guardrail 11: Source Endpoints Have Engine-Specific Prerequisites
Each source engine has unique requirements. Oracle: supplemental logging, ARCHIVELOG mode for CDC. MySQL: binlog enabled, server_id set. PostgreSQL: logical replication slot, wal_sender_timeout. SQL Server: MS-CDC enabled on database and tables. MongoDB: oplog access. Always check engine-specific documentation before starting.

## Guardrail 12: Endpoint SSL/TLS Configuration Varies by Engine
SSL/TLS settings differ between source and target engines. Some require certificate import, others use built-in CA. SslMode options: none, require, verify-ca, verify-full. Using verify-ca or verify-full requires uploading the CA certificate. RDS endpoints support SSL by default but DMS must be configured to use it.
