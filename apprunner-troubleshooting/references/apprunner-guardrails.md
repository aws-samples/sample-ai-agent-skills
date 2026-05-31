# AWS App Runner Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any App Runner issue.

## Guardrail 1: Source Type Differences — ECR Image vs GitHub Source Code
App Runner has two fundamentally different source types. ECR image sources (ImageRepository) use a pre-built container image — there is no build phase. GitHub/Bitbucket code sources (CodeRepository) include a build phase where App Runner builds the container. The configuration, error patterns, and troubleshooting steps differ significantly. Always confirm the source type before investigating build or deployment issues.

## Guardrail 2: Auto-Scaling Is Based on Concurrent Requests, Not CPU/Memory
App Runner scales based on the number of concurrent requests per instance (MaxConcurrency, default 100). It does NOT scale based on CPU utilization or memory usage like ECS or EC2 Auto Scaling. MinSize defaults to 1 and cannot be set to 0 — there is always at least one warm instance. MaxSize defaults to 25. Never recommend CPU-based scaling policies for App Runner.

## Guardrail 3: Health Check Failures Cause Automatic Deployment Rollback
When a new deployment fails health checks, App Runner automatically rolls back to the previous working version. HTTP health checks require the application to return 200 on the configured path. TCP health checks only verify the port is open. Default settings: interval 5s, timeout 2s, healthy threshold 1, unhealthy threshold 5. A misconfigured health check path is the most common cause of deployment rollbacks.

## Guardrail 4: Custom Domains Use App Runner-Managed Certificates, Not ACM
App Runner provisions and manages its own SSL/TLS certificates for custom domains. You do NOT create or import ACM certificates for App Runner custom domains. Domain validation requires adding CNAME records provided by App Runner to your DNS. Validation can take up to 48 hours. The default *.awsapprunner.com domain always has SSL enabled automatically.

## Guardrail 5: VPC Connector Is Required for Private Resource Access
By default, App Runner services can only reach public internet endpoints. To access resources in a VPC (RDS, ElastiCache, internal ALBs), you must attach a VPC connector specifying subnets and security groups. When a VPC connector is attached, outbound internet traffic for VPC-routed destinations goes through the VPC (requires NAT gateway for internet access from VPC subnets). Public endpoint access still works directly without NAT.

## Guardrail 6: Environment Variables Are Not Encrypted at Rest in Configuration
Environment variables set on an App Runner service are visible in the service configuration via the API and console. For sensitive values, use secrets that reference SSM Parameter Store parameters or Secrets Manager secrets. The instance role must have permission to read the referenced secrets. Never store credentials directly in environment variables.

## Guardrail 7: Build Phase vs Runtime Phase Have Different Contexts
For code-based sources, the build phase runs build commands (e.g., npm install, pip install) with build environment variables. The runtime phase runs the start command with runtime environment variables. Build-time environment variables are NOT available at runtime and vice versa. Build timeout defaults to 10 minutes. Build failures and runtime failures have different log streams.

## Guardrail 8: Deployment Triggers — Manual vs Automatic Have Different Behaviors
Manual deployment requires an explicit StartDeployment API call or console action. Automatic deployment triggers on every push to the configured branch (GitHub) or every image push to the configured ECR repository/tag. Automatic deployments can cause unexpected updates if the branch or tag receives frequent pushes. Pausing auto-deployment does not affect manual deployments.

## Guardrail 9: Instance Role Trust Policy Must Allow apprunner.amazonaws.com
The instance role provides AWS credentials to the running application. The role's trust policy must include apprunner.amazonaws.com as a trusted principal. Without this, the role cannot be attached to the service. This is different from ECS (ecs-tasks.amazonaws.com) or Lambda (lambda.amazonaws.com). Always verify the trust policy when instance role errors occur.

## Guardrail 10: X-Ray Daemon Is Managed by App Runner — Do Not Run It Yourself
When X-Ray tracing is enabled via an observability configuration, App Runner automatically runs the X-Ray daemon as a sidecar. The application only needs to use the X-Ray SDK or OpenTelemetry SDK to send traces. Do NOT include the X-Ray daemon in your container image or start command. Duplicate daemons cause port conflicts and trace loss.

## Guardrail 11: No SSH, Exec, or Direct Container Access
App Runner does NOT support any form of direct container access — no SSH, no exec, no shell. This is by design for security. All debugging must be done through CloudWatch Logs, CloudWatch Metrics, X-Ray traces, and application-level instrumentation. Never suggest SSH or exec as a troubleshooting step for App Runner.

## Guardrail 12: Application Must Listen on the PORT Environment Variable
App Runner sets the PORT environment variable (default 8080) and routes traffic to that port. The application MUST read the PORT environment variable and listen on it. If the application hardcodes a different port (e.g., 3000) and ignores PORT, health checks will fail because App Runner sends health checks to the configured port. Always verify the application respects the PORT env var.
