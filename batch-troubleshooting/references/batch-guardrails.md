# AWS Batch Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS Batch issue.

## Guardrail 1: RUNNABLE Means Waiting for Resources, Not Running
Jobs in RUNNABLE state are waiting for compute capacity to become available. This is NOT the same as RUNNING. Common causes: compute environment not scaling, instance types unavailable, resource requests exceeding CE capacity, VPC/subnet issues, or service quotas reached. Check CE status and scaling activity.

## Guardrail 2: EC2 and Fargate Have Different Capabilities
EC2 compute environments support: GPUs, custom AMIs, placement groups, EFA, all instance types, and multi-node parallel jobs. Fargate is serverless with limits: max 16 vCPU, 120 GB memory, no GPU, no multi-node parallel, no custom AMI. Fargate Spot is available. Choose based on workload requirements.

## Guardrail 3: Exit Codes Indicate Specific Failure Types
Exit code 0 = success. Exit code 1 = general application error. Exit code 137 = OOM killed (SIGKILL, container exceeded memory limit). Exit code 139 = segmentation fault. Exit code 143 = SIGTERM (graceful termination). Always check container logs for the specific error. Exit code 137 requires increasing memory allocation.

## Guardrail 4: Spot Interruptions Are Expected, Not Failures
Spot instances can be reclaimed by AWS with 2-minute warning. Batch automatically retries Spot-interrupted jobs if retry strategy is configured. Spot interruptions are normal and expected. Use retry strategies with appropriate attempt counts. Mix Spot and On-Demand for critical workloads. Spot allocation strategy affects interruption frequency.

## Guardrail 5: Array Jobs Are Independent Child Jobs
Each array child job runs independently with its own AWS_BATCH_JOB_ARRAY_INDEX (0 to size-1). Failed children do not affect other children. Array size can be 2-10,000. All children share the same job definition. Dependencies can be set between array jobs (SEQUENTIAL or N_TO_N). Monitor individual child job status.

## Guardrail 6: Multi-Node Parallel Requires EC2 Compute
Multi-node parallel jobs are NOT supported on Fargate. They require EC2 compute environments. The main node (index 0) coordinates work. All nodes must communicate via the network. ECS task placement ensures nodes are scheduled. Node count is fixed at submission time. Use placement groups for low-latency communication.

## Guardrail 7: Job Queue Priority Determines Evaluation Order
Higher priority values mean higher priority. Queues are evaluated in priority order. Jobs within a queue are FIFO by default. Fair-share scheduling can override priority-based ordering. A queue can be associated with multiple compute environments (evaluated in order). Disabled queues do not schedule jobs.

## Guardrail 8: Resource Requests Must Match CE Instance Capabilities
Job vCPU and memory requests must be satisfiable by at least one instance type in the CE. Requesting 64 vCPU when the largest instance type has 48 vCPU causes RUNNABLE stuck. GPU requests require GPU instance types (p3, p4, g4, g5). Fargate has fixed vCPU/memory combinations. Always verify resource requests against CE capabilities.

## Guardrail 9: Compute Environment Scaling Has Limits and Delays
Managed CEs scale based on job queue demand. Scaling from 0 to first instance takes 2-5 minutes. maxvCpus limits total CE capacity. minvCpus keeps instances warm (costs money). desiredvCpus is managed by Batch. Scaling is not instant — plan for startup latency. Unmanaged CEs require manual scaling.

## Guardrail 10: Job Retry Strategies Handle Transient Failures
Retry strategies specify attempts (1-10) and optional exit code-based evaluation. evaluateOnExit can RETRY or EXIT based on exit code, status reason, or action. Spot interruptions can be automatically retried. Each retry is a new attempt with a new container. Retry does not preserve container state.

## Guardrail 11: Fair-Share Scheduling Distributes Resources Across Users
Fair-share scheduling policies distribute compute resources based on share identifiers. Each share has a weight and optional share decay. Jobs are scheduled to maintain fair distribution, not strict priority. Fair-share overrides queue priority ordering. Configure share identifiers in job submissions.

## Guardrail 12: Managed vs Unmanaged Compute Environments
Managed CEs: Batch manages instance lifecycle, scaling, and patching. You specify instance types, VPC, and limits. Unmanaged CEs: You manage instances manually and register them with the ECS cluster. Managed is recommended for most use cases. Unmanaged provides full control but requires more operational effort.
