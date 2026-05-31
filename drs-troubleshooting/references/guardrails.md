# AWS Elastic Disaster Recovery Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS DRS issue.

## Guardrail 1: Agent Must Be Installed Before Source Server Appears
The AWS Replication Agent must be installed on each source server before it appears in DRS. DRS does not auto-discover servers. The agent requires outbound TCP 443 to DRS endpoints and TCP 1500 to the replication server. Verify agent status on the source server before investigating DRS-side issues.

## Guardrail 2: Replication Servers Are Managed — Do Not Terminate
DRS automatically creates replication server EC2 instances in the staging area subnet. These instances appear in the EC2 console with DRS-specific tags. Do NOT manually terminate, stop, or modify replication servers. DRS manages their lifecycle automatically.

## Guardrail 3: Initial Sync Duration Depends on Data Volume
Initial replication sync can take hours to days depending on total disk size and available bandwidth. A source server showing INITIAL_SYNC state is NOT broken — it is performing first-time data copy. Check dataReplicationInfo.etaDateTime for estimated completion. Only investigate if the state is STALLED or DISCONNECTED.

## Guardrail 4: Staging Area Subnet Requires Specific Connectivity
The staging area subnet must have outbound internet access (via NAT Gateway or internet gateway) OR VPC endpoints for S3 and DRS APIs. Without this connectivity, replication data cannot be written to S3 and API calls fail. Verify route tables and security groups for the staging area subnet.

## Guardrail 5: Launch Configuration Is Per Source Server
Launch configuration (instance type, subnet, security groups) is set per source server, not globally. Changes to the launch configuration template do NOT retroactively update existing source servers. You must update each source server's launch configuration individually or use the API to batch update.

## Guardrail 6: Drill Instances Must Be Cleaned Up
Drill (test) instances launched via StartRecovery with isDrill=true create real EC2 instances that incur costs. These instances are NOT automatically terminated after the drill. You must explicitly clean up drill instances. Failing to do so results in ongoing EC2 charges.

## Guardrail 7: Failback Requires Failback Client Installation
Failback from AWS to the original source requires installing the Failback Client on the recovery instance. The Failback Client replicates data back to the original source infrastructure. Failback is not automatic — it requires explicit initiation and monitoring.

## Guardrail 8: Recovery Instance Boot Issues May Be OS-Level
After a successful recovery launch, the recovery instance may fail to boot correctly due to OS-level issues (driver incompatibility, network configuration, boot loader). These are not DRS failures — they are post-launch OS configuration issues. Check EC2 instance console output and system logs.

## Guardrail 9: Replication Bandwidth Affects Source Server Performance
DRS replication consumes network bandwidth on the source server. High replication throughput can impact source server application performance. Bandwidth throttling is configurable in the replication configuration. Do not assume unlimited bandwidth is always appropriate.

## Guardrail 10: Source Server Lifecycle States Have Specific Meanings
Source server lifecycle states (NOT_READY, READY_FOR_TEST, READY_FOR_CUTOVER, CUTOVER, DISCONNECTED) indicate the server's position in the DR workflow. Each state has specific allowed actions. Do not attempt operations that are invalid for the current state.

## Guardrail 11: Data Replication Lag Is Not Always a Problem
Small replication lag (seconds to minutes) is normal during active write operations on the source. Lag becomes concerning only when it consistently grows or exceeds RPO requirements. Check lagDuration in dataReplicationInfo and compare against the defined RPO.

## Guardrail 12: DRS Costs Include Replication Servers and Storage
DRS costs include replication server instances, EBS volumes for staging, S3 storage for snapshots, and data transfer. Recovery and drill instances incur standard EC2 costs. Do not overlook the cost of running replication infrastructure continuously.
