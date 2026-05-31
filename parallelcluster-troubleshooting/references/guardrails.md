# AWS ParallelCluster Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any ParallelCluster issue.

## Guardrail 1: CloudFormation Stack Events Contain the Real Error
ParallelCluster uses CloudFormation for cluster provisioning. When creation or update fails, the actual error is in the CloudFormation stack events, not the pcluster CLI output. Always check `pcluster get-cluster-stack-events` or `aws cloudformation describe-stack-events` for the root cause.

## Guardrail 2: Slurm Is the Only Supported Scheduler in ParallelCluster 3.x
ParallelCluster 3.x exclusively supports Slurm. SGE (Sun Grid Engine) and Torque were supported in ParallelCluster 2.x but have been removed. Do not suggest SGE or Torque configurations. All scheduler commands use Slurm syntax (sinfo, squeue, sbatch, scontrol).

## Guardrail 3: FSx for Lustre Scratch Filesystems Are NOT Persistent
FSx for Lustre scratch filesystems (SCRATCH_1, SCRATCH_2) are temporary. Data is lost when the filesystem is deleted, which happens when the cluster is deleted. Only PERSISTENT_1 and PERSISTENT_2 deployment types retain data. Always verify the deployment type before advising on data persistence.

## Guardrail 4: Compute Nodes Are Dynamic — Not Pre-Provisioned
ParallelCluster compute nodes are launched on demand by Slurm's power-saving plugin (clustermgtd). They are NOT pre-provisioned EC2 instances. Nodes are launched when jobs are submitted and terminated after the idle timeout (ScaledownIdletime). Do not expect compute nodes to be running when no jobs are queued.

## Guardrail 5: Custom AMIs Must Be Built with pcluster build-image
Custom AMIs for ParallelCluster must be built using `pcluster build-image`. Using arbitrary AMIs (even Amazon Linux 2) will fail because ParallelCluster requires specific software, configurations, and Slurm installation. Never suggest using a standard AMI directly.

## Guardrail 6: Head Node Is a Single Point of Failure
The head node is a single EC2 instance running the Slurm controller, scheduler, and management daemons. If it fails, the entire cluster is unavailable. There is no built-in head node HA. Recommend EBS snapshots, CloudWatch alarms, and instance recovery for resilience.

## Guardrail 7: Shared Storage Mount Points Are Fixed
EFS and FSx filesystems are mounted at specific paths defined in the cluster configuration. The default shared directory is /shared. Changing mount points requires cluster update or recreation. Do not suggest manually mounting filesystems — use the cluster configuration.

## Guardrail 8: Multi-Queue Maps to Slurm Partitions
Each queue in the ParallelCluster configuration maps to a Slurm partition. Different queues can have different instance types, scaling settings, and networking. Jobs are submitted to specific partitions with `sbatch -p <partition>`. Do not confuse ParallelCluster queues with SQS queues.

## Guardrail 9: Head Node Needs Internet or VPC Endpoints
The head node requires internet access (via NAT gateway or internet gateway) or VPC endpoints for AWS API calls, package installation, and CloudWatch metrics. Without connectivity, cluster operations fail silently. Always verify head node network connectivity.

## Guardrail 10: Job Accounting Requires sacct Configuration
Slurm job accounting (sacct) requires the accounting storage to be configured. ParallelCluster configures Slurm accounting by default with a local database. For persistent accounting across cluster recreations, use an external database.

## Guardrail 11: Scaling Has Cooldown and Launch Delays
Compute node scaling is not instant. Node launch involves EC2 instance provisioning, boot, Slurm registration, and health check. This can take 2-5 minutes. Scale-down has an idle timeout (default 10 minutes). Do not expect immediate scaling response.

## Guardrail 12: Many Cluster Settings Are Immutable After Creation
Some cluster configuration settings cannot be changed via update and require cluster recreation. These include VPC, subnet, scheduler type, and some storage configurations. Always check the ParallelCluster documentation for which settings support update vs require recreation.
