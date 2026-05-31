---
name: apprunner-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS App Runner problems
  by analyzing service configurations, deployment status, auto-scaling settings,
  health checks, networking, and following structured runbooks.
  Activate when: deployment failures, build failures, image pull errors,
  health check failures, application crashes, port configuration issues,
  auto-scaling problems, cold start latency, custom domain/SSL issues,
  VPC connector problems, outbound connectivity failures, ECR source issues,
  GitHub source issues, instance role permission errors, secrets/environment
  variable problems, logging issues, X-Ray tracing configuration, or the
  user says something is wrong with App Runner without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with App Runner, ECR, IAM, CloudWatch,
  CloudTrail, ACM, Route 53, and optionally VPC and X-Ray permissions.
---

# App Runner Diagnostics

## When to use

Any App Runner investigation where the console alone is insufficient — deployment debugging, build troubleshooting, runtime crash analysis, scaling configuration, networking issues, source repository problems, security configuration, or observability setup.

## Investigation workflow

### Step 1 — Collect and triage

```
aws apprunner list-services --query 'ServiceSummaryList[*].{Name:ServiceName,Arn:ServiceArn,Status:Status}'
version: "1.0.0"
last_updated: "2025-04-12"
aws apprunner describe-service --service-arn <service-arn>
aws apprunner list-operations --service-arn <service-arn> --max-results 10
aws cloudwatch get-metric-statistics --namespace AWS/AppRunner --metric-name 2xxStatusResponses --dimensions Name=ServiceName,Value=<service-name> Name=ServiceId,Value=<service-id> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws cloudwatch get-metric-statistics --namespace AWS/AppRunner --metric-name 5xxStatusResponses --dimensions Name=ServiceName,Value=<service-name> Name=ServiceId,Value=<service-id> --start-time <start> --end-time <end> --period 300 --statistics Sum
```

### Step 2 — Domain deep dive

```
aws apprunner describe-service --service-arn <service-arn> --query 'Service.SourceConfiguration'
aws apprunner describe-service --service-arn <service-arn> --query 'Service.HealthCheckConfiguration'
aws apprunner describe-auto-scaling-configuration --auto-scaling-configuration-arn <asc-arn>
aws apprunner list-auto-scaling-configurations --query 'AutoScalingConfigurationSummaryList[*].{Name:AutoScalingConfigurationName,Arn:AutoScalingConfigurationArn}'
version: "1.0.0"
last_updated: "2025-04-12"
aws apprunner describe-service --service-arn <service-arn> --query 'Service.NetworkConfiguration'
aws apprunner list-custom-domains --service-arn <service-arn>
```

### Step 3 — Detailed investigation

```
aws apprunner describe-service --service-arn <service-arn> --query 'Service.InstanceConfiguration'
aws apprunner list-vpc-connectors --query 'VpcConnectors[*].{Name:VpcConnectorName,Arn:VpcConnectorArn,Status:Status}'
version: "1.0.0"
last_updated: "2025-04-12"
aws apprunner describe-vpc-connector --vpc-connector-arn <vpc-connector-arn>
aws apprunner describe-observability-configuration --observability-configuration-arn <obs-arn>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=apprunner.amazonaws.com --max-results 10
aws ecr describe-images --repository-name <repo-name> --query 'imageDetails[*].{Tags:imageTags,Pushed:imagePushedAt,Size:imageSizeInBytes}'
```

Read `references/apprunner-guardrails.md` before concluding on any App Runner issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `list-services` | List all App Runner services |
| `describe-service` | Full service details including source, health check, networking |
| `list-operations` | Recent deployment and operation history |
| `describe-auto-scaling-configuration` | Auto-scaling min/max instances and concurrency |
| `list-custom-domains` | Custom domain and certificate validation status |
| `describe-vpc-connector` | VPC connector subnets and security groups |
| `describe-observability-configuration` | X-Ray tracing configuration |
| `list-vpc-connectors` | Available VPC connectors |
| `ecr describe-images` | ECR image details for source verification |
| `acm describe-certificate` | Certificate status for custom domains |
| `list-connections` | GitHub/Bitbucket connection status |
| `describe-custom-domains` | Domain validation records |

## Gotchas: App Runner

- Source types: App Runner supports two source types — ECR image (pre-built container) and GitHub source code (built by App Runner). The configuration differs significantly. ECR image sources use ImageRepository; code sources use CodeRepository with build configuration. Mixing up the source type leads to confusing errors.
- Auto-scaling min/max instances: MinSize defaults to 1 (always warm). Setting MinSize to 0 is NOT supported — there is always at least 1 instance running. MaxSize defaults to 25. MaxConcurrency (requests per instance) defaults to 100. Scaling is based on concurrent requests, not CPU/memory.
- Health check configuration: App Runner supports TCP and HTTP health checks. HTTP health checks require a path (default /). The default interval is 5 seconds, timeout is 2 seconds, healthy threshold is 1, unhealthy threshold is 5. If the health check fails, the deployment rolls back automatically.
- Custom domain with ACM certificate: App Runner provisions its own SSL certificate for custom domains — you do NOT bring your own ACM certificate. You must validate the domain by adding CNAME records provided by App Runner. Certificate validation can take up to 48 hours. The default *.awsapprunner.com domain always has SSL.
- VPC connector for private resources: By default, App Runner services can only access public endpoints. To access private resources (RDS, ElastiCache, etc.), you must attach a VPC connector. The VPC connector specifies subnets and security groups. Outbound traffic to the internet still works without a NAT gateway for public endpoints, but VPC-routed traffic requires proper routing.
- Environment variables and secrets: Environment variables are set at service creation/update and are available to the application at runtime. Secrets can reference SSM Parameter Store or Secrets Manager ARNs. The instance role must have permission to read the secrets. Environment variables are NOT encrypted at rest in the App Runner configuration.
- Build vs runtime configuration: For code-based sources, App Runner has separate build and runtime phases. Build commands run during deployment. Start command runs the application. Build environment has different resource limits than runtime. Build timeout defaults to 10 minutes.
- Deployment triggers: Manual deployment requires explicit API call or console action. Automatic deployment triggers on every push to the configured branch (GitHub) or every image push (ECR). Automatic deployments can be paused/resumed. Each deployment creates a new operation.
- Instance role for AWS service access: The instance role is an IAM role assumed by the running application. It provides credentials for AWS SDK calls. Without an instance role, the application cannot access AWS services. The role must have a trust policy allowing apprunner.amazonaws.com to assume it.
- Observability (X-Ray tracing): X-Ray tracing requires an observability configuration attached to the service. The application must use the X-Ray SDK or OpenTelemetry to instrument traces. App Runner automatically configures the X-Ray daemon — you do not run it yourself.
- No SSH/exec into containers: App Runner does NOT support SSH, exec, or any direct access to running containers. Debugging must be done through logs (CloudWatch), metrics, and application-level instrumentation. There is no equivalent of ECS Exec or kubectl exec.
- Port must be configurable via PORT env var: App Runner sets the PORT environment variable and expects the application to listen on it. The default port is 8080. If the application ignores the PORT env var and hardcodes a different port, health checks will fail and the service will not become healthy.

## Anti-hallucination rules

1. Always cite specific service configurations, operation statuses, or API responses as evidence.
2. App Runner does NOT support SSH/exec into containers. Never suggest connecting to a running container directly.
3. Custom domains use App Runner-managed certificates, NOT user-provided ACM certificates. Never tell users to create their own ACM certificate for App Runner custom domains.
4. MinSize cannot be 0 — there is always at least 1 instance. Never suggest scaling to zero.
5. Auto-scaling is based on concurrent requests, NOT CPU or memory utilization. Never claim CPU-based scaling.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 24 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Deployment | A1-A3 | Deployment failures, build failures, image pull errors |
| B — Runtime | B1-B3 | Health check failures, application crashes, port configuration |
| C — Scaling | C1-C2 | Auto-scaling issues, cold start latency |
| D — Networking | D1-D3 | Custom domain/SSL, VPC connector, outbound connectivity |
| E — Source | E1-E2 | ECR source issues, GitHub source issues |
| F — Security | F1-F2 | Instance role permissions, secrets/env vars |
| G — Observability | G1-G2 | Logging issues, X-Ray tracing |
| Z — Catch-All | Z1 | General troubleshooting |
