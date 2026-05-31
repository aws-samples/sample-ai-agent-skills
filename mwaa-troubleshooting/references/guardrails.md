# Amazon MWAA Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon MWAA issue.

## Guardrail 1: MWAA Is a Managed Service — No Server-Level Access
Amazon MWAA is a fully managed Apache Airflow service. You cannot SSH into workers, schedulers, or web servers. There is no access to airflow.cfg directly. Configuration overrides use environment variables with the `AIRFLOW__SECTION__KEY` prefix set through the MWAA console or API. Never suggest connecting to Airflow components directly or modifying configuration files on disk.

## Guardrail 2: DAGs Must Reside in S3 — No Local File System
MWAA syncs DAGs exclusively from an S3 bucket. You cannot place DAGs on a local file system, use git-sync, or mount volumes. DAG files must be uploaded to the configured S3 DAG path. The scheduler periodically syncs from S3 — changes are not instant. DAG parse errors appear in the DAGProcessing log group, not at upload time.

## Guardrail 3: VPC Requires Private Subnets with NAT Gateway
MWAA environments must be deployed in private subnets with outbound internet access via NAT gateway (or VPC endpoints for AWS services). Public subnets are not supported. Without NAT or VPC endpoints, the environment cannot install PyPI packages, access AWS APIs, or reach external services. Always verify NAT gateway status when troubleshooting connectivity.

## Guardrail 4: Execution Role Is the Single IAM Identity
The MWAA execution role is used for ALL AWS operations: S3 DAG bucket access, CloudWatch log delivery, Secrets Manager reads, KMS decryption, and any AWS service calls from DAG tasks. If any AWS operation fails with access denied, check the execution role first. There is no per-DAG or per-task IAM role — everything uses the execution role.

## Guardrail 5: requirements.txt Failures Can Break the Entire Environment
Package installation from requirements.txt runs at environment startup. If any package fails to install (version conflict, missing dependency, network issue), the environment can enter an unhealthy state. Always pin package versions, test locally with the same Python/Airflow version, and use constraints files. Check the requirements installation log before investigating other issues.

## Guardrail 6: plugins.zip Must Follow Airflow Plugin Structure
The plugins.zip file is extracted to the Airflow plugins directory. Custom operators, hooks, sensors, and macros must follow Airflow's expected directory structure. An incorrectly structured ZIP will silently fail to load plugins. The ZIP root should contain the plugin files directly (not nested in an extra directory).

## Guardrail 7: Web Server Access Mode Determines Connectivity
MWAA supports PUBLIC_ONLY (internet-facing) or PRIVATE_ONLY (VPC-only) web server access. Public mode creates a public endpoint; private mode requires a VPC endpoint (com.amazonaws.<region>.airflow.api) in the same VPC. Switching modes requires an environment update. If the web UI is inaccessible, verify the access mode and network path first.

## Guardrail 8: Worker Auto-Scaling Is Not Instant
Worker scaling is based on task queue depth with configurable min/max workers. Scaling up takes several minutes as new Fargate tasks launch. During scale-up, tasks queue and may appear stuck. Do not confuse scaling delay with a broken scheduler. Check the RunningTasks and QueuedTasks CloudWatch metrics to distinguish scaling lag from actual failures.

## Guardrail 9: Secrets Backend Requires Specific Naming Conventions
When using Secrets Manager or Systems Manager Parameter Store as the secrets backend, connection IDs and variables must follow the configured prefix pattern (default: `airflow/connections/` and `airflow/variables/`). Incorrect naming causes silent lookup failures where Airflow falls back to the metastore database. Always verify the prefix configuration and secret names.

## Guardrail 10: CloudWatch Log Groups Are Per-Component
MWAA creates separate CloudWatch log groups for each component: Scheduler, Worker, WebServer, DAGProcessing, and Task. Each can be independently enabled or disabled at different log levels (CRITICAL, ERROR, WARNING, INFO, DEBUG). Missing logs may simply mean that log group is disabled, not that the component is broken.

## Guardrail 11: Environment Updates Take 20-30 Minutes
MWAA environment updates (configuration changes, version upgrades, requirements/plugins updates) are not instant. The environment enters UPDATING status and may be partially unavailable. Do not interpret slow updates as failures. Monitor the environment status and check CloudTrail for update events. Concurrent updates are not supported.

## Guardrail 12: Airflow Version Upgrades Require DAG Compatibility Testing
In-place Airflow version upgrades may break DAGs that use deprecated operators, hooks, or import paths. Always test DAGs against the target Airflow version before upgrading. Check the Airflow migration guide for breaking changes. Rollback from a failed upgrade may require creating a new environment with the previous version.
