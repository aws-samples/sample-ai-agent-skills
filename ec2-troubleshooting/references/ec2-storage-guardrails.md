# EC2 Storage Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any EC2 storage/EBS issue.

## Guardrail 1: Stop+Start is NOT Reboot
Stop+start migrates the instance to new host hardware and LOSES instance store data. Reboot keeps the same host and preserves instance store data. Never confuse these operations. Stop+start also changes the public IPv4 address (unless EIP is attached).

## Guardrail 2: Instance Store is Ephemeral
Instance store data is LOST on: stop, terminate, host failure, hibernation. Data survives ONLY on reboot. This is by design, not a bug. Never claim instance store data can be recovered after stop/terminate.

## Guardrail 3: EBS Root Volume Delete-on-Termination
Root volumes created from AMIs default to DeleteOnTermination=true. Additional volumes default to false. Check this BEFORE terminating an instance if you need the data.

## Guardrail 4: gp2 vs gp3 Baseline Performance
gp2 baseline: 3 IOPS/GB (min 100, max 16000), burst to 3000 IOPS. A 100GB gp2 volume has only 300 baseline IOPS.
gp3 baseline: 3000 IOPS and 125 MB/s regardless of size. gp3 is almost always better than gp2.

## Guardrail 5: EBS-Optimized Bandwidth is Instance-Level
EBS-optimized throughput is shared across ALL attached volumes. A small instance (t3.micro: 87.5 MB/s) will bottleneck even a high-IOPS volume. Check instance EBS bandwidth, not just volume specs.

## Guardrail 6: NVMe Device Names Don't Match Block Device Mapping
On Nitro instances, /dev/sda1 in the block device mapping appears as /dev/nvme0n1 in the OS. Use `lsblk`, `nvme id-ctrl`, or `ebsnvme-id` to map NVMe devices to EBS volume IDs.

## Guardrail 7: EBS Multi-Attach Requires Cluster Filesystem
io1/io2 Multi-Attach allows a volume to be attached to up to 16 Nitro instances. But you MUST use a cluster-aware filesystem (GFS2, OCFS2). Using ext4 or xfs with Multi-Attach WILL corrupt data.

## Guardrail 8: Volume Modification Cooldown
After modifying a volume (size, type, IOPS), there is a 6-hour cooldown before the next modification. Plan modifications carefully. The modification itself may take hours to complete (volume enters 'optimizing' state).

## Guardrail 9: Snapshot Restore Performance
First read to each block of a volume restored from a snapshot incurs latency (lazy loading from S3). Use Fast Snapshot Restore (FSR) or pre-warm by reading all blocks (`dd if=/dev/xvdf of=/dev/null bs=1M`) for consistent performance.

## Guardrail 10: io1/io2 IOPS-to-Size Ratio
io1: max 50 IOPS per GB. io2: max 500 IOPS per GB. A 100GB io1 volume can have max 5000 IOPS. A 100GB io2 volume can have max 50000 IOPS. Requesting more than the ratio allows will fail.

## Guardrail 11: EBS Encryption is Transparent but Requires KMS
Encrypted volumes use KMS for every I/O operation. If the KMS key is disabled/deleted or permissions are revoked, the volume becomes unusable. This can happen silently — the volume appears attached but all I/O fails.
