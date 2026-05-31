# Amazon FSx Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon FSx issue.

## Guardrail 1: Lustre Scratch Has NO Data Replication
Lustre scratch file systems do NOT replicate data. If the underlying hardware fails, data is lost. Scratch is designed for temporary, high-performance workloads where data can be regenerated from S3 or recomputed. Persistent Lustre replicates data within the same AZ and supports automatic failover to replacement servers. Neither scratch nor persistent supports multi-AZ. Never assume scratch Lustre data survives hardware failure.

## Guardrail 2: Windows FSx REQUIRES Active Directory
FSx for Windows File Server cannot be created without joining an Active Directory domain — either AWS Managed Microsoft AD or a self-managed AD. The file system uses AD for authentication and authorization. DNS resolution to domain controllers must work from the FSx subnets. AD connectivity issues are the most common cause of Windows FSx creation failures.

## Guardrail 3: ONTAP Multi-Protocol Shares Same Data
FSx for ONTAP supports NFS, SMB, and iSCSI simultaneously on the same underlying data. A single volume can be accessed via multiple protocols. Permission mapping between UNIX and Windows styles is handled by the SVM's name mapping configuration. Misconfigured name mapping causes access denied errors when switching protocols. Always verify name mapping rules when troubleshooting cross-protocol access.

## Guardrail 4: OpenZFS Is NFS Only
FSx for OpenZFS supports only NFS (v3, v4.0, v4.1, v4.2) for data access. It does NOT support SMB or iSCSI. Never suggest SMB shares or iSCSI LUNs for OpenZFS file systems. For Windows client access to OpenZFS, an NFS client must be installed on the Windows host or a gateway solution must be used.

## Guardrail 5: Storage Capacity Cannot Be Decreased
Storage capacity scaling is one-way — you can increase but NEVER decrease the storage capacity of any FSx file system type. For Lustre, capacity increases must be in increments of the original storage capacity. For ONTAP, the total SSD storage can be increased. Plan capacity carefully as over-provisioning cannot be reversed without creating a new file system.

## Guardrail 6: Lustre Data Repository Association Links to S3
Lustre DRA (Data Repository Association) links the file system to an S3 bucket. Data is lazy-loaded on first access (not pre-loaded unless an import task is run). Export tasks push modified data back to S3. Auto-import keeps file system metadata in sync with S3 changes. DRA failures are commonly caused by S3 bucket policy restrictions, missing IAM role permissions, or S3 bucket in a different region.

## Guardrail 7: Windows Multi-AZ Uses Floating IP for Failover
Windows FSx multi-AZ deployment uses an active-standby model with automatic failover. A floating IP address moves from the active to the standby file server during failover. DNS records are updated automatically. Failover typically completes within 30-60 seconds. Single-AZ deployments have NO automatic failover — if the file server fails, it must be restored from backup.

## Guardrail 8: ONTAP SVMs Provide Workload Isolation
Each SVM (Storage Virtual Machine) on an ONTAP file system has its own DNS name, network interfaces, and protocol configuration. SVMs provide multi-tenancy — different applications or teams can use separate SVMs on the same file system. SVM administration can be delegated. Deleting an SVM deletes all its volumes. A file system can have multiple SVMs but each consumes management resources.

## Guardrail 9: Throughput Modes Differ by File System Type
Lustre throughput is determined by storage capacity and deployment type (50/100/200 MB/s per TiB). Windows FSx offers throughput capacity that can be changed independently of storage. ONTAP throughput capacity is per file system and can be scaled. OpenZFS throughput is set at creation and can be updated. Never assume throughput scales automatically with storage — it depends on the file system type.

## Guardrail 10: Backup Compatibility Varies by File System Type
All FSx types support AWS Backup integration, but native backup behavior differs. Lustre backups are only supported for persistent file systems (not scratch). Windows FSx supports both automatic daily backups and user-initiated backups. ONTAP supports volume-level snapshots and file system backups. OpenZFS supports snapshots and backups. Backup retention and scheduling differ by type.

## Guardrail 11: Encryption at Rest Is Set at Creation and Immutable
All FSx file systems are encrypted at rest using KMS keys specified at creation time. The encryption key CANNOT be changed after creation. The default is the AWS-managed FSx key. For customer-managed keys, the KMS key policy must grant FSx the required permissions. Cross-region operations require KMS keys in the destination region.

## Guardrail 12: Network Interfaces Are Managed by FSx
FSx creates and manages Elastic Network Interfaces (ENIs) in the specified subnets. These ENIs must not be deleted or modified manually. Security groups attached to the ENIs control network access. Required ports differ by type: Lustre (988), Windows (445, 5985), ONTAP (111, 635, 2049, 4046, 11104, 11105), OpenZFS (111, 2049). Incorrect security group rules are a common cause of mount failures.
