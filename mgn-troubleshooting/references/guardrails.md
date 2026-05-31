# AWS Application Migration Service Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS MGN issue.

## Guardrail 1: Agent Is Shared with DRS
The AWS Replication Agent is shared between MGN and DRS. A source server can only be registered with ONE service at a time. If a server is registered with DRS, it cannot simultaneously be used with MGN. Check which service the server is registered with before troubleshooting.

## Guardrail 2: FinalizeCutover Is Irreversible
Calling FinalizeCutover permanently stops replication for the source server. After finalization, you cannot resume replication or roll back through MGN. The source server moves to CUTOVER state. Only finalize after thorough validation of the migrated workload.

## Guardrail 3: Cutover Creates New EC2 Instances
Each cutover attempt creates new EC2 instances from the latest replication snapshot. Previous cutover instances are NOT automatically terminated. Multiple cutover attempts result in multiple running instances, all incurring costs.

## Guardrail 4: Waves and Applications Are Organizational Only
Waves and Applications in MGN are grouping constructs for managing migration at scale. They do NOT affect replication behavior, launch configuration, or cutover operations. Servers replicate independently regardless of wave/application assignment.

## Guardrail 5: Post-Launch Actions Require SSM
Post-launch actions execute via AWS Systems Manager Run Command. The migrated instance must have SSM agent installed, an IAM instance profile with SSM permissions, and network connectivity to SSM endpoints. Without these, post-launch actions silently fail.

## Guardrail 6: Launch Template Changes Don't Auto-Apply
Changes to launch configuration templates only affect NEW source servers. Existing source servers retain their individual launch configurations. Update each server individually with: aws mgn update-launch-configuration --source-server-id <id>.

## Guardrail 7: Replication Uses Staging Area Resources
MGN creates replication servers, EBS volumes, and S3 storage in the staging area. These resources incur costs even when no migration is actively happening. Decommission source servers from MGN when migration is complete to stop staging area costs.

## Guardrail 8: Source Server Lifecycle States Gate Operations
Source server states (NOT_READY, READY_FOR_TEST, READY_FOR_CUTOVER, CUTOVER, DISCONNECTED) determine which operations are allowed. You cannot cutover a server in NOT_READY state. You cannot test a server in CUTOVER state. Always check state before attempting operations.

## Guardrail 9: Initial Sync Must Complete Before Testing
Testing or cutover cannot proceed until initial replication sync completes. Initial sync duration depends on data volume and bandwidth. Check dataReplicationInfo.dataReplicationState — it must be CONTINUOUS before testing.

## Guardrail 10: Cross-Account Migration Requires Additional Setup
Migrating to a different AWS account requires cross-account IAM roles and proper staging area configuration in the target account. The replication agent credentials must have permissions in the target account.

## Guardrail 11: Disk Replication Is Block-Level
MGN replicates at the block level, not file level. All data on included disks is replicated, including OS, applications, and data. You cannot selectively replicate specific files or directories. Disk exclusion is at the whole-disk level only.

## Guardrail 12: Network Configuration Changes After Migration
Migrated instances get new IP addresses, MAC addresses, and network interfaces in AWS. Applications with hardcoded IPs or MAC-based licensing need post-migration configuration. DNS updates may be required to redirect traffic.
