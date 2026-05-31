# EFS Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any EFS issue.

## Guardrail 1: General Purpose Is Default and Recommended
General Purpose performance mode is recommended for most workloads. Only consider Max I/O if PercentIOLimit is consistently above 95%. Max I/O has higher latency.

## Guardrail 2: Mount Targets Are Per-AZ
Each AZ needs its own mount target. Instances should use the mount target in their AZ. Cross-AZ mounting works but adds latency and data transfer costs.

## Guardrail 3: Security Group Must Allow NFS Port 2049
The mount target security group must allow inbound TCP on port 2049 from the client's security group or CIDR. The client's security group must allow outbound to port 2049.

## Guardrail 4: Burst Credits Deplete on Small File Systems
Bursting throughput is 50 MiB/s per TB of storage. A 100 GB file system only gets 5 MiB/s baseline. Burst credits deplete quickly under sustained load. Check BurstCreditBalance metric.

## Guardrail 5: Encryption at Rest Cannot Be Added Later
Encryption at rest must be enabled when creating the file system. There is no way to encrypt an existing unencrypted file system. You must create a new encrypted file system and migrate data.

## Guardrail 6: Access Points Override Client Credentials
Access points enforce a specific POSIX user ID and group ID regardless of what the NFS client sends. This is by design for security. Do not confuse access point enforcement with permission errors.

## Guardrail 7: EFS CSI Driver Requires Mount Targets in Node Subnets
The EFS CSI driver mounts from the worker node. If there is no mount target in the node's AZ/subnet, the PVC will be stuck in Pending. Always create mount targets in all AZs where nodes run.

## Guardrail 8: Lifecycle IA Minimum File Size Is 128 KB
Files smaller than 128 KB are not transitioned to Infrequent Access storage. They remain in Standard storage class regardless of lifecycle policy.

## Guardrail 9: Replication Destination Is Read-Only
EFS replication creates a read-only destination file system. It cannot be written to until you perform a failover. Failover promotes the destination to read-write and breaks replication.

## Guardrail 10: Performance Mode Cannot Be Changed
Performance mode (General Purpose or Max I/O) is set at creation and cannot be changed. To switch modes, create a new file system and migrate data.

## Guardrail 11: Elastic Throughput Has Higher Per-Request Cost
Elastic throughput mode automatically scales but charges per GiB transferred. For predictable workloads, Provisioned throughput is more cost-effective.

## Guardrail 12: In-Transit Encryption Requires Mount Helper
Encryption in transit requires mounting with the amazon-efs-utils mount helper using the -o tls option. Standard NFS mount commands do not encrypt traffic.
