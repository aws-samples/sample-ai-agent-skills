# AWS Storage Gateway Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS Storage Gateway issue.

## Guardrail 1: Cache Disks Are Required for ALL Gateway Types
Every Storage Gateway type requires local cache disks. File gateways cache recently accessed files. Volume gateways (cached mode) cache frequently accessed data blocks. Tape gateways cache data before uploading to S3. Cache disks should be SSD (not HDD) for acceptable performance. Minimum cache disk size is 150 GB. Insufficient cache is the #1 cause of gateway performance issues.

## Guardrail 2: Gateway VM Has Minimum Resource Requirements
File gateway: minimum 4 vCPUs, 16 GB RAM, 80 GB root disk. Volume/Tape gateway: minimum 4 vCPUs, 16 GB RAM, 80 GB root disk. Cache disk and upload buffer are separate from root disk. Undersized VMs cause gateway crashes, poor performance, and data loss risk. For production workloads, allocate more resources than the minimum.

## Guardrail 3: File Shares Map Files to S3 Objects
Each file on a file gateway share becomes an S3 object. The S3 key is the file path relative to the share root. File metadata (permissions, timestamps) is stored as S3 object metadata. Directories become zero-byte S3 objects with a trailing slash. Changes made directly to S3 are NOT visible through the file share until a cache refresh is performed.

## Guardrail 4: Activation Requires Port 80 Temporarily
Gateway activation uses HTTP (port 80) to retrieve an activation key. This port is only needed during the activation process. After activation, close port 80. Normal gateway operation uses port 443 (HTTPS) for communication with AWS. The gateway also uses ports 1026-1028, 1031, and 2222 for local management console access.

## Guardrail 5: Volume Gateway Mode Cannot Be Changed After Creation
Cached volumes store primary data in S3 with local cache. Stored volumes store primary data locally with S3 backup. The mode is set when creating each volume and CANNOT be changed afterward. To switch modes, you must create a new volume and migrate data. Choose the mode carefully based on latency and capacity requirements.

## Guardrail 6: Virtual Tape Size and Count Limits
Individual virtual tapes range from 100 GB to 5 TB. A single tape gateway supports up to 1,500 virtual tapes with a total of 1 PB. Tapes must be ejected from the virtual tape library before archiving to S3 Glacier or S3 Glacier Deep Archive. Retrieving archived tapes takes 3-5 hours (Glacier) or 12 hours (Deep Archive). Do not confuse tape library with tape archive.

## Guardrail 7: Cache Refresh Is NOT Automatic for S3 Changes
When files are added or modified directly in S3 (bypassing the gateway), the file share cache does NOT automatically update. You must trigger a cache refresh: `aws storagegateway refresh-cache --file-share-arn <arn>`. Scheduled cache refresh can be configured. Without refresh, the gateway serves stale data. Cache refresh can be slow for large buckets.

## Guardrail 8: AD Integration Requires DNS and Time Sync
SMB file shares with Active Directory authentication require: (1) gateway can resolve AD domain via DNS, (2) gateway can reach AD domain controllers on required ports (389, 445, 88, etc.), (3) time difference between gateway and AD is less than 5 minutes. Join domain with: `aws storagegateway join-domain`. Failed AD joins are usually DNS or time sync issues.

## Guardrail 9: Upload Buffer Is NOT the Same as Cache Disk
Upload buffer is temporary storage for data being uploaded to AWS. Cache disk stores frequently accessed data locally. Both are required and must be separate disks. Upload buffer minimum is 150 GB. When upload buffer is full, the gateway stops accepting writes. Monitor CachePercentDirty and UploadBufferPercentUsed metrics.

## Guardrail 10: Gateway Maintenance Window Causes Brief Unavailability
Each gateway has a weekly maintenance window for software updates. During maintenance, the gateway is briefly unavailable (typically minutes). Schedule maintenance during low-usage periods. Check maintenance time: `aws storagegateway describe-maintenance-start-time`. Maintenance cannot be skipped but can be rescheduled.

## Guardrail 11: Bandwidth Rate Limiting Applies to Upload and Download Separately
Bandwidth limits can be set independently for upload (gateway to AWS) and download (AWS to gateway). Limits are in bits per second, not bytes. Minimum limit is 51200 bps (50 Kbps). Setting limits too low causes upload buffer to fill and writes to stall. Check limits: `aws storagegateway describe-bandwidth-rate-limit`.

## Guardrail 12: Choose Gateway Type Based on Use Case
File Gateway: NFS/SMB access to S3 storage. Volume Gateway: iSCSI block storage backed by S3. Tape Gateway: virtual tape library backed by S3/Glacier. You cannot change a gateway's type after deployment. Each type has different cache, network, and resource requirements. Deploying the wrong type requires redeployment.
