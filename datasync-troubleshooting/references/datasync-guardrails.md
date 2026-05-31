# AWS DataSync Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS DataSync issue.

## Guardrail 1: Agent Requires Outbound Connectivity on Specific Ports
DataSync agents need outbound access on port 443 (HTTPS) for data transfer and service communication, and port 80 (HTTP) only during initial activation. If using VPC endpoints, the agent communicates through the VPC endpoint instead of public endpoints. Blocked ports are the most common cause of agent connectivity failures. Check security groups, NACLs, and firewalls.

## Guardrail 2: Task Verification Modes Have Specific Behaviors
ONLY_FILES_TRANSFERRED (default) verifies only files that were transferred in the current execution. POINT_IN_TIME_CONSISTENT verifies all files at source and destination, which is thorough but slow for large datasets. NONE skips verification entirely. Do not confuse these modes or invent new ones. Verification failures do not mean the transfer failed — they mean a mismatch was detected.

## Guardrail 3: Metadata Preservation Varies by Location Type
POSIX metadata (uid, gid, permissions, timestamps) is preserved for NFS, EFS, and FSx for Lustre. NTFS metadata (ACLs, timestamps, attributes) is preserved for SMB and FSx for Windows. When transferring to S3, metadata is stored as S3 object metadata and user-defined tags. Not all metadata types transfer between all location type combinations. Check the DataSync documentation for your specific source-destination pair.

## Guardrail 4: Filters Use Glob Patterns, NOT Regex
Include and exclude filters use glob-style patterns (e.g., *.log, /folder/*). They do NOT support regular expressions. Excludes are evaluated first, then includes. An empty include filter means "include everything not excluded." Filters apply to relative paths from the source location subdirectory. The pattern /temp/* excludes the temp directory at the root of the source, not nested temp directories.

## Guardrail 5: Bandwidth Throttling Is Per-Task Execution
The BytesPerSecond option limits bandwidth for a single task execution, not for the agent or account. Multiple concurrent task executions on the same agent share the agent's network bandwidth independently of per-task throttling. To limit total agent bandwidth, you must manage the number of concurrent tasks. A value of -1 means no throttling.

## Guardrail 6: Service-Linked Role Is NOT the S3 IAM Role
DataSync automatically creates the AWSServiceRoleForDataSyncDataAccess service-linked role. This is separate from the IAM role you specify for S3 locations. The S3 location IAM role must have s3:GetBucketLocation, s3:ListBucket, s3:GetObject, s3:PutObject, and related permissions. Do not confuse the service-linked role with the S3 bucket access role.

## Guardrail 7: Incremental Transfers Compare Source and Destination
By default, subsequent task executions only transfer files that have changed (new, modified, or deleted). DataSync compares file metadata (size, timestamps) to detect changes. The comparison happens during the Prepare phase. If you need a full transfer every time, configure the task to overwrite all files. Deleted file handling depends on the PreserveDeletedFiles option.

## Guardrail 8: Agent VM Has Minimum Resource Requirements
The DataSync agent VM requires at least 4 vCPUs, 16 GB RAM, and 80 GB disk space. Undersized agents cause poor transfer performance, task failures, or agent crashes. For transferring millions of small files, allocate more resources (especially RAM). The agent VM must run on VMware ESXi, Microsoft Hyper-V, Linux KVM, or as an EC2 instance.

## Guardrail 9: Cross-Account Transfers Require IAM Roles in Both Accounts
For cross-account transfers, you need an IAM role in the destination account that the DataSync service can assume. The source account's DataSync task assumes this cross-account role. Both accounts must have proper trust policies configured. Cross-account transfers also require the destination resource (S3 bucket, EFS, FSx) to allow access from the cross-account role.

## Guardrail 10: Location Subdirectory Affects Transfer Scope
When configuring a location, the subdirectory parameter defines the root of the transfer. For NFS: /export/data means DataSync accesses /export/data on the NFS server. For S3: /prefix/ means DataSync operates within that S3 prefix. Changing the subdirectory changes which files are in scope for transfer. An incorrect subdirectory is a common cause of "no files transferred."

## Guardrail 11: Task Execution Lifecycle States Determine Behavior
Task executions go through states: QUEUED → LAUNCHING → PREPARING → TRANSFERRING → VERIFYING → SUCCESS/ERROR. A task stuck in LAUNCHING usually indicates agent issues. Stuck in PREPARING means source/destination scanning is slow (large file count). TRANSFERRING duration depends on data volume and bandwidth. You cannot skip states or force transitions.

## Guardrail 12: NFS/SMB Mount Permissions Must Allow DataSync Access
For NFS locations, the DataSync agent must be able to mount the NFS export. The NFS server must allow the agent's IP in its export configuration. For SMB locations, valid credentials (user/password) with read/write access are required. NFS version (3 or 4.x) must match the server configuration. Mount failures appear as location validation errors.
