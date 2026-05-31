# Amazon Managed Service for Apache Flink Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Managed Flink issue.

## Guardrail 1: API Namespace Is Still kinesisanalyticsv2
The service was renamed from Kinesis Data Analytics to Managed Service for Apache Flink, but the API namespace remains `kinesisanalyticsv2`. All CLI commands use `aws kinesisanalyticsv2`. Do not search for a `managed-flink` CLI namespace.

## Guardrail 2: Checkpoints and Savepoints Are Different
Checkpoints are automatic, periodic state snapshots for fault tolerance. Savepoints (application snapshots) are manual, user-triggered state captures for upgrades and rollbacks. Checkpoints are managed by Flink; savepoints are managed by the user via the API.

## Guardrail 3: Parallelism Has Two Components
Total parallelism = Parallelism × ParallelismPerKPU. Parallelism sets the number of parallel tasks. ParallelismPerKPU sets how many parallel tasks run per Kinesis Processing Unit. Changing one without understanding the other causes resource miscalculation.

## Guardrail 4: VPC Applications Cannot Access Internet by Default
Applications configured with VPC connectivity run inside your VPC subnets. They cannot access the internet or public AWS endpoints by default. They need NAT Gateway for internet access and VPC endpoints for AWS services like Kinesis, S3, and CloudWatch.

## Guardrail 5: Application Snapshots Are Required for Safe Updates
Before updating application code or configuration, create an application snapshot. Without a snapshot, you cannot roll back to the previous state. Snapshots capture the full Flink state including operator state and keyed state.

## Guardrail 6: Runtime Environment Cannot Be Changed After Creation
The Flink runtime version (FLINK-1_15, FLINK-1_18, etc.) is set at application creation and cannot be changed. To upgrade the Flink version, you must create a new application, restore from a snapshot, and delete the old application.

## Guardrail 7: CloudWatch Metrics Have Different Granularities
Some Managed Flink metrics are reported per-application, others per-task, and others per-operator. Aggregating metrics at the wrong level produces misleading results. Check the metric dimensions before drawing conclusions.

## Guardrail 8: Source and Sink Connectors Have Different Error Patterns
Source connector errors (cannot read data) and sink connector errors (cannot write data) have different root causes. Source errors are often permission or stream configuration issues. Sink errors are often serialization or destination capacity issues.

## Guardrail 9: Studio Notebooks Have Different Resource Limits
Studio notebooks (Zeppelin-based) have different memory limits, execution timeouts, and supported operations compared to code-based applications. Notebooks are designed for interactive development, not production workloads.

## Guardrail 10: Application Restarts Reset to Last Successful Checkpoint
When an application restarts (manually or due to failure), it resumes from the last successful checkpoint. If checkpointing is disabled or failing, the application restarts from scratch, potentially reprocessing or losing data.

## Guardrail 11: KPU Pricing Is Based on Running Time
Kinesis Processing Units (KPUs) are billed per hour of running time. Increasing parallelism increases KPU count and cost. Auto-scaling can increase costs unexpectedly. Monitor KPU usage and set appropriate scaling limits.

## Guardrail 12: Application Logs Require Explicit Configuration
Application logging to CloudWatch is not enabled by default. You must configure log stream ARN in the application's monitoring configuration. Without logging, troubleshooting Flink job failures is extremely difficult.
