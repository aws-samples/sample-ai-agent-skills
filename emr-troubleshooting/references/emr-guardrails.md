# Amazon EMR Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon EMR issue.

## Guardrail 1: Cluster Logs Are in S3, Not CloudWatch by Default
EMR writes logs to the configured S3 log bucket at s3://<log-bucket>/<cluster-id>/. This includes step logs, application logs (Spark, Hive, etc.), and node logs. CloudWatch logging must be enabled separately. Always check the S3 log bucket first for debugging.

## Guardrail 2: Bootstrap Actions Run Before Applications Start
Bootstrap actions execute on all nodes before Hadoop/Spark applications are installed. If a bootstrap action fails, the cluster terminates. Bootstrap actions have a default timeout. They cannot modify application configurations that haven't been installed yet. Debug by checking s3://<log-bucket>/<cluster-id>/node/<node-id>/bootstrap-actions/.

## Guardrail 3: YARN Controls All Application Resource Allocation
YARN (Yet Another Resource Negotiator) manages CPU and memory for all applications on EMR. Spark executors, Hive queries, and other applications request resources from YARN. Misconfigured YARN settings (yarn.nodemanager.resource.memory-mb, yarn.scheduler.maximum-allocation-mb) cause OOM errors or underutilization. Always check YARN configuration.

## Guardrail 4: S3 Has Strong Read-After-Write Consistency Natively
Since December 2020, Amazon S3 provides strong read-after-write consistency for all operations. EMRFS consistent view is deprecated and should NOT be configured on new clusters. Do not recommend EMRFS consistency workarounds — they are unnecessary and add overhead.

## Guardrail 5: Managed Scaling Has Cooldown and Limits
Managed scaling adds/removes instances based on YARN metrics. It has a cooldown period between scaling actions. Minimum and maximum units must be configured. Scaling decisions consider both pending YARN containers and cluster utilization. Scaling is not instant — instance provisioning takes minutes.

## Guardrail 6: EMR on EKS Runs on Existing EKS Clusters
EMR on EKS creates virtual clusters on existing EKS clusters. It requires: (1) EKS cluster with EMR namespace, (2) virtual cluster registration, (3) execution role with EKS and S3 permissions. Debugging requires checking both EMR job logs and EKS pod logs. Pod scheduling failures are common.

## Guardrail 7: EMR Serverless Auto-Provisions Resources
EMR Serverless does not require cluster management. Applications define resource limits (max CPU, memory). Pre-initialized capacity can reduce cold start. Job failures often relate to IAM permissions, S3 access, or exceeding resource limits. Check application and job run status separately.

## Guardrail 8: Security Configurations Are Immutable After Creation
Encryption settings, Kerberos configuration, and Lake Formation integration are set at cluster creation via security configurations. These CANNOT be changed on a running cluster. To change security settings, create a new cluster with the updated security configuration.

## Guardrail 9: Instance Fleets Select from Multiple Instance Types
Instance fleets allow specifying multiple instance types. EMR selects based on availability and price. On-Demand and Spot instances can be mixed. Spot interruptions cause task node loss (core nodes use On-Demand by default). Spot interruptions do NOT cause data loss on HDFS if core nodes are On-Demand.

## Guardrail 10: Spark Memory Model Has Multiple Components
Spark memory is divided into: executor memory (spark.executor.memory), memory overhead (spark.executor.memoryOverhead), and off-heap memory. Total memory per executor = executor memory + overhead. OOM errors can occur in any component. The default overhead is max(384MB, 0.10 * executor memory).

## Guardrail 11: Termination Protection Prevents Accidental Deletion
Termination protection prevents the cluster from being terminated by API calls or console actions. It must be explicitly disabled before termination. Auto-termination (idle timeout) is a separate feature. Termination protection does NOT prevent termination due to bootstrap action failures or unrecoverable errors.

## Guardrail 12: Step Concurrency Controls Parallel Execution
EMR supports concurrent step execution (configurable, default 1). Steps can run in parallel up to the concurrency level. CANCEL_AND_WAIT action type cancels running steps. Step failures can be configured to continue or terminate the cluster. Check step action-on-failure setting.
