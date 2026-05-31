# AWS Deadline Cloud Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Deadline Cloud issue.

## Guardrail 1: Farm → Fleet/Queue → Job Hierarchy
Deadline Cloud uses a strict hierarchy: Farms contain Fleets and Queues. Fleets provide compute workers. Queues organize and schedule jobs. Jobs contain tasks executed in sessions on workers. Always identify the correct level in the hierarchy when troubleshooting.

## Guardrail 2: Service-Managed vs Customer-Managed Fleets
Service-managed fleets (SMF) are auto-scaled by AWS. Customer-managed fleets (CMF) use your EC2 instances with the worker agent installed. Troubleshooting differs significantly: SMF issues involve service configuration; CMF issues involve EC2 and agent setup.

## Guardrail 3: Worker Agent Is Separate from Worker Host
The worker agent is software running on the host that communicates with Deadline Cloud. The worker host is the EC2 instance or on-premises machine. Agent failures (software crashes, authentication errors) are different from host failures (instance termination, resource exhaustion).

## Guardrail 4: Job Attachments Use S3
Job attachments transfer files between submitter and workers via S3. This is NOT a shared file system. Files are uploaded to S3 on submission and downloaded to workers at session start. Large attachments increase job startup time. Storage profiles map S3 paths to local file system paths.

## Guardrail 5: Budgets Do Not Stop Running Jobs
Budget limits prevent NEW job submissions when exhausted. They do NOT terminate or pause running jobs. Running jobs continue to completion even after the budget is exceeded. Budget monitoring is informational for running workloads.

## Guardrail 6: Queue-Fleet Associations Control Job Routing
Jobs submitted to a queue can only run on fleets associated with that queue. If no fleet is associated, jobs remain pending. Multiple fleets can be associated with one queue for capacity. Verify associations when jobs are stuck in PENDING.

## Guardrail 7: License Endpoints Require Network Connectivity
License endpoints connect workers to license servers for commercial DCC applications (Maya, Houdini, etc.). Workers must have network access to the license server. VPC configuration, security groups, and DNS resolution must all be correct.

## Guardrail 8: Custom Job Templates Use OpenJD Specification
Custom job templates follow the Open Job Description (OpenJD) specification. Template validation happens at submission time, not at template creation. Syntax errors in templates cause submission failures with validation error messages.

## Guardrail 9: Storage Profiles Are Queue-Specific
Storage profiles are associated with specific queues and define how file paths are mapped between the submitter's file system and the worker's file system. Different OS workers (Linux/Windows) may need different storage profiles for the same queue.

## Guardrail 10: Monitor Requires IAM Identity Center
Deadline Cloud monitors (web portal) require AWS IAM Identity Center (formerly SSO) for user authentication. Without Identity Center configured, the monitor cannot authenticate users. This is separate from IAM roles used by workers and the API.

## Guardrail 11: Fleet Scaling Has Warm-Up Time
Service-managed fleets take time to scale up. New instances must launch, boot, install the worker agent, and register with the service. This warm-up period means jobs may wait several minutes before workers are available. Do not assume instant scaling.

## Guardrail 12: Job Lifecycle States Are Sequential
Jobs progress through states: CREATE_IN_PROGRESS → CREATE_COMPLETE → PENDING → RUNNING → SUCCEEDED/FAILED/CANCELED. Jobs cannot skip states. Understanding the current state helps identify where in the pipeline the issue occurs.
