# AWS Elastic Beanstalk Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Elastic Beanstalk issue.

## Guardrail 1: .ebextensions Execute in Alphabetical Order
Files in the `.ebextensions/` directory are processed in lexicographic (alphabetical) order. A file named `01_packages.config` runs before `02_services.config`. If one extension depends on resources created by another, the naming order is critical. A failure in any extension file causes the entire deployment to fail. Use numeric prefixes to enforce ordering.

## Guardrail 2: Environment Rebuild vs Update Are Fundamentally Different
`rebuild-environment` terminates ALL instances and recreates the environment from scratch using the current configuration. `update-environment` modifies the existing environment in place. Rebuild loses all instance-local data, resets the environment ID, and creates new resources. Use update for configuration changes; rebuild only when the environment is corrupted or needs a clean slate.

## Guardrail 3: Platform Branch vs Platform Version
A platform branch (e.g., "Python 3.9 running on 64bit Amazon Linux 2") is a family of platform versions. A platform version is a specific dated release within a branch. Managed platform updates move between versions within the SAME branch. Changing branches (e.g., Amazon Linux 2 to Amazon Linux 2023) requires a blue/green deployment or environment rebuild — in-place branch changes are not supported.

## Guardrail 4: Immutable Deployment Creates a Temporary ASG
During immutable deployments, Beanstalk creates a SEPARATE temporary Auto Scaling group with new instances running the new version. If health checks pass, the new instances are moved to the original ASG and old instances are terminated. If health checks fail, the entire temporary ASG is deleted. This temporarily DOUBLES your instance count and associated costs.

## Guardrail 5: Worker Tier Uses SQS Polling, NOT Direct HTTP
Worker environments do NOT receive direct HTTP traffic from the internet. The Beanstalk worker daemon polls an SQS queue for messages and POSTs them to `http://localhost/` on the instance. The daemon handles message visibility, retry logic, and dead-letter queue routing. Misconfiguring the SQS queue URL, permissions, or the local application endpoint are common failure causes.

## Guardrail 6: Enhanced Health Requires the Health Agent
Enhanced health reporting requires the Elastic Beanstalk health agent running on each instance. On Amazon Linux 2 and later platforms, the agent is installed automatically. On older platforms or custom AMIs, the agent may be missing. Without the agent, only basic health (ELB health check pass/fail) is available. Enhanced health provides application-level metrics, request latency, and HTTP status code breakdowns.

## Guardrail 7: Managed Platform Updates Can Break Applications
Managed platform updates automatically apply new platform versions during a configured maintenance window. New versions may include runtime upgrades (e.g., Python 3.9 → 3.11), dependency changes, or configuration defaults that break existing applications. Always test platform updates in a staging environment. Use immutable update type for managed updates to enable automatic rollback on failure.

## Guardrail 8: Environment Cloning Does NOT Copy Data
Cloning an environment creates a new environment with the same configuration, platform, and application version. It does NOT copy RDS data, S3 objects, EFS files, or any persistent state. If the source environment has a coupled RDS instance, the clone gets a new EMPTY database. Decoupled RDS instances are not cloned at all — the clone has no database unless configured separately.

## Guardrail 9: Docker Platform Configuration Differs from Non-Docker
Docker environments use `Dockerrun.aws.json` (v1 for single container, v3 for ECS-based multi-container) or a `Dockerfile`. Non-Docker environments use `Procfile`, platform hooks (`platform/hooks/`), and buildfiles. Mixing these configuration approaches causes deployment failures. Multi-container Docker (v3) runs on ECS and requires an ECS task definition format in `Dockerrun.aws.json`.

## Guardrail 10: leader_only Runs on Exactly One Instance
Commands marked with `leader_only: true` in `.ebextensions` run on only one instance — the leader (first instance launched in the ASG). This is useful for one-time tasks like database migrations. However, if the leader instance is replaced (terminated and relaunched), the command runs again on the new leader. leader_only does NOT guarantee idempotency — the command must be written to handle re-execution safely.

## Guardrail 11: Deployment Policy Selection Has Trade-offs
All-at-once is fastest but causes downtime. Rolling updates a batch at a time but reduces capacity. Rolling with additional batch maintains full capacity but takes longer. Immutable is safest (full rollback) but doubles instances temporarily. Traffic splitting (canary) routes a percentage to new instances. Blue/green creates an entirely new environment. Never assume one policy fits all — choose based on availability requirements and rollback needs.

## Guardrail 12: Environment Tier Cannot Be Changed After Creation
An environment is created as either a web server tier or a worker tier. The tier CANNOT be changed after creation. To switch from web to worker (or vice versa), you must create a new environment with the desired tier and migrate your application. The two tiers have fundamentally different architectures — web tier has an ELB, worker tier has an SQS queue and daemon.
