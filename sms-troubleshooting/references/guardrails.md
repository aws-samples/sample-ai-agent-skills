# AWS Server Migration Service Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any SMS issue.

## Guardrail 1: SMS Is Legacy — Recommend MGN for New Migrations
AWS Server Migration Service is being replaced by Application Migration Service (MGN). For new migrations, recommend MGN. SMS is still supported for existing migrations but new features go to MGN.

## Guardrail 2: SMS Connector Must Be Deployed and Healthy
The SMS Connector is a FreeBSD VM deployed in the source environment (VMware/Hyper-V). It must be running and connected to both the hypervisor and AWS. Without a healthy connector, no operations work.

## Guardrail 3: Replication Jobs Have a 90-Day Limit
Each replication job has a maximum duration of 90 days. After 90 days, the job expires and must be recreated. Plan migration cutover within this window.

## Guardrail 4: Replication Is Incremental After First Run
The first replication run copies the full disk. Subsequent runs are incremental, copying only changed blocks. This makes ongoing replication faster but the first run takes longest.

## Guardrail 5: AMI Generation Follows Replication Run
AMIs are created from completed replication run snapshots. AMI generation is a separate step that happens after the replication data is in AWS. Failed replication runs don't produce AMIs.

## Guardrail 6: Application Migration Groups Servers Logically
Application migration in SMS groups multiple servers for coordinated migration. It doesn't guarantee atomic migration — each server replicates independently. Coordination is for launch ordering.

## Guardrail 7: Connector Needs vCenter/SCVMM Access
The SMS Connector needs API access to VMware vCenter or Microsoft SCVMM to discover and replicate VMs. Credentials must be configured in the connector.

## Guardrail 8: S3 Bucket Is Used for Replication Data
SMS uses an S3 bucket to temporarily store replication data before creating EBS snapshots. The bucket must be accessible and have sufficient space.

## Guardrail 9: IAM Role Required for SMS Operations
SMS needs an IAM role (ServerMigrationServiceRole) with permissions for EC2, S3, and SMS operations. The role must trust sms.amazonaws.com.

## Guardrail 10: Server Catalog Comes from Connector
The server catalog in SMS is populated by the connector from the hypervisor inventory. If servers are missing, check connector connectivity to vCenter/SCVMM.

## Guardrail 11: Replication Frequency Affects RPO
Replication frequency (1-24 hours) determines the Recovery Point Objective. More frequent replication means less data loss but more resource usage on the source.

## Guardrail 12: Launch Configuration Defines Target Instance
The launch configuration specifies instance type, VPC, subnet, security groups for the migrated server. It must be configured before launching the migrated instance.
