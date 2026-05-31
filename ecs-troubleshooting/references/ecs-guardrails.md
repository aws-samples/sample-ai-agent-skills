# ECS Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any ECS issue.

## Guardrail 1: Task Role ≠ Execution Role
Execution role (executionRoleArn): used by ECS AGENT for pulling images, pushing logs, retrieving secrets. Task role (taskRoleArn): used by the APPLICATION for AWS API calls (DynamoDB, S3, etc.). Never confuse these. Wrong role = wrong fix.

## Guardrail 2: Exit Code 137 is Not Always OOM
Exit code 137 = SIGKILL. This can be OOM kill, manual task stop, or service scaling down. Always check stoppedReason for "OutOfMemoryError" to confirm OOM. Don't assume 137 = OOM without checking.

## Guardrail 3: Fargate Needs NAT or VPC Endpoints
Fargate tasks in private subnets have NO internet access by default. They need NAT gateway or VPC endpoints for: ECR image pulls (ecr.api + ecr.dkr + s3), CloudWatch Logs, Secrets Manager, and any external APIs.

## Guardrail 4: awsvpc Mode = One ENI Per Task
In awsvpc network mode, each task gets its own ENI. On EC2 launch type, this is limited by the instance's ENI limit. Enable ENI trunking to increase density. Fargate always uses awsvpc.

## Guardrail 5: Hard Memory Limit vs Soft Limit
memory = hard limit (OOM kill threshold). memoryReservation = soft limit (scheduling reservation only, container can use more). On Fargate, task-level memory is the hard limit shared across all containers.

## Guardrail 6: Service Events Are Limited to 100
ECS service events only retain the last 100 entries. Older events are permanently lost. Check events immediately when investigating. Don't assume you can look at events from days ago.

## Guardrail 7: Fargate Platform Versions Matter
Platform version 1.4.0 added: EFS support, ephemeral storage configuration, ECS Exec, increased ENI density. Using LATEST resolves to the latest version. Pinning to an old version misses features and fixes.

## Guardrail 8: Container Health Check ≠ ELB Health Check
ECS container health checks (in task definition) and ELB target health checks are independent. Both must pass for a task to receive traffic. Failing either one causes issues but for different reasons.

## Guardrail 9: Deployment Circuit Breaker Behavior
Circuit breaker monitors consecutive task launch failures. After threshold, it automatically rolls back to the last successful deployment. This is a safety net, not a deployment strategy. Investigate WHY it triggered.

## Guardrail 10: Bridge Mode Port Conflicts
In bridge network mode, host ports can conflict. Dynamic port mapping (hostPort=0) avoids conflicts but requires ALB for service discovery. Host mode gives direct host port access but limits one task per port per instance.

## Guardrail 11: Fargate assignPublicIp Requirement
Fargate tasks in PUBLIC subnets need assignPublicIp=ENABLED to get a public IP for internet access. In PRIVATE subnets, this setting is irrelevant — use NAT gateway instead.

## Guardrail 12: ECS Exec Prerequisites
ECS Exec requires ALL of: enableExecuteCommand=true on service, task role with ssmmessages permissions, Fargate platform 1.4.0+, SSM Session Manager plugin installed locally, and VPC endpoint for ssmmessages (if private subnet).
