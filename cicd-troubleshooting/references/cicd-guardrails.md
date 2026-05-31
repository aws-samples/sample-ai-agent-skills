# AWS CI/CD Pipeline Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any CI/CD pipeline issue.

## Guardrail 1: Pipeline Executions Can Be Superseded
When multiple source changes trigger a pipeline, CodePipeline does NOT queue them. If a new execution reaches a stage where an older execution is still in progress, the older execution is SUPERSEDED — it stops, and the newer execution takes over. This is expected behavior, not a failure. Check `aws codepipeline list-pipeline-executions` and look for SUPERSEDED status before investigating "failed" executions.

## Guardrail 2: CodeBuild Timeout Has Specific Limits
CodeBuild default timeout is 60 minutes. Maximum is 8 hours (480 minutes). The timeout applies to the entire build, not individual phases. If a build consistently times out, check `timeoutInMinutes` in the project configuration: `aws codebuild batch-get-projects --names <project>`. Queued time does NOT count toward the build timeout — only active build time does.

## Guardrail 3: CodeDeploy In-Place vs Blue/Green Are Fundamentally Different
In-place deployments update existing instances (one at a time or in batches). Blue/green deployments create entirely new instances, shift traffic, then terminate old instances. Rollback behavior differs: in-place rollback redeploys the previous revision; blue/green rollback reroutes traffic back to the original instances. Never apply in-place troubleshooting steps to blue/green deployments or vice versa.

## Guardrail 4: Artifact Encryption Requires Customer-Managed KMS for Cross-Account
CodePipeline encrypts artifacts in S3 using the default `aws/s3` AWS-managed key OR a customer-managed KMS key. The default key CANNOT be shared across accounts. Cross-account pipelines MUST use a customer-managed KMS key with a key policy granting access to the target account. Always check `aws codepipeline get-pipeline --query 'pipeline.artifactStore.encryptionKey'` for cross-account issues.

## Guardrail 5: Cross-Account Pipelines Need Three Things
Cross-account pipeline access requires ALL three: (1) Customer-managed KMS key with cross-account key policy, (2) S3 artifact bucket policy granting cross-account access, (3) IAM role in the target account with trust policy allowing the pipeline account to assume it. Missing any one causes "Access Denied" errors that can be hard to pinpoint. Check all three systematically.

## Guardrail 6: CodeBuild Local Cache Is Not Guaranteed
CodeBuild offers three cache modes: SOURCE_CACHE, DOCKER_LAYER_CACHE, and CUSTOM_CACHE. All local caches are stored on the build host. Builds may run on different hosts, so cache hits are NOT guaranteed. Docker layer cache requires privileged mode. For reliable caching, use S3 cache instead of local cache. Never assume a cache miss is a bug — it's expected behavior.

## Guardrail 7: buildspec.yml post_build Phase Always Runs
buildspec.yml phases execute in order: install → pre_build → build → post_build. If any phase fails, subsequent phases are SKIPPED — except post_build, which ALWAYS runs. During post_build after a failure, `CODEBUILD_BUILD_SUCCEEDING` is set to "0". Use post_build for cleanup tasks. Never put critical build logic in post_build assuming it only runs on success.

## Guardrail 8: appspec.yml Hook Order Is Deployment-Type Specific
EC2/On-Premises hooks: BeforeInstall → AfterInstall → ApplicationStart → ValidateService. Lambda hooks: BeforeAllowTraffic → AfterAllowTraffic. ECS hooks: BeforeInstall → AfterInstall → AfterAllowTestTraffic → BeforeAllowTraffic → AfterAllowTraffic. Hook scripts must exit with code 0 for success. Any non-zero exit code fails the deployment at that lifecycle event. Always check which deployment type before advising on hook order.

## Guardrail 9: CodeDeploy Rollback Triggers Require ALARM State
Automatic rollback can be triggered by deployment failure OR CloudWatch alarm breach. However, alarms in INSUFFICIENT_DATA state do NOT trigger rollbacks — only ALARM state does. If rollbacks are not firing when expected, check the alarm state: `aws cloudwatch describe-alarms --alarm-names <alarm>`. Also verify the deployment group has auto-rollback enabled: `aws codedeploy get-deployment-group --query 'deploymentGroupInfo.autoRollbackConfiguration'`.

## Guardrail 10: Source Detection Methods Have Different Characteristics
Polling: CodePipeline checks for changes every ~1 minute. Adds latency, generates API calls. Webhooks: near-instant detection via HTTP callback from the source provider. EventBridge: recommended for CodeCommit and S3 sources, uses CloudWatch Events rules. CodeStar Connections: used for GitHub/Bitbucket, manages OAuth and webhooks automatically. Never assume all source types use the same detection mechanism.

## Guardrail 11: CodeBuild in VPC Needs Internet Access for AWS Services
When a CodeBuild project is configured with VPC access, the build environment is placed in the specified subnets. Private subnets need a NAT gateway or VPC endpoints to reach AWS services (S3 for artifacts, ECR for images, CloudWatch for logs). Without internet access, builds fail at DOWNLOAD_SOURCE or when pulling Docker images. Always check VPC configuration when builds fail with network-related errors.

## Guardrail 12: CodeDeploy Minimum Healthy Hosts Controls Deployment Pace
The deployment configuration's `minimumHealthyHosts` setting determines how many instances must remain in service during deployment. If set too high (e.g., 90%) and some instances fail health checks, the deployment gets stuck — it cannot take enough instances out of service to proceed. If set too low, too many instances go offline simultaneously. Check `aws codedeploy get-deployment-config --deployment-config-name <config>` to understand the deployment pace constraints.
