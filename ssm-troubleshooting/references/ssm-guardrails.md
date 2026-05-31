# AWS Systems Manager Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS Systems Manager issue.

## Guardrail 1: SSM Agent Requires BOTH Installation AND IAM Role
The SSM Agent must be installed on the instance AND the instance must have an IAM instance profile with the AmazonSSMManagedInstanceCore managed policy (or equivalent custom permissions). Without both, the instance will not appear as a managed instance. Amazon Linux 2, Amazon Linux 2023, and Windows Server 2016+ AMIs come with SSM Agent pre-installed, but still require the IAM role.

## Guardrail 2: Three SSM Endpoints Are Required for Full Functionality
SSM Agent requires outbound HTTPS (443) access to three endpoints: ssm.<region>.amazonaws.com (for API calls), ssmmessages.<region>.amazonaws.com (for Session Manager), and ec2messages.<region>.amazonaws.com (for Run Command). Missing any one endpoint causes partial functionality loss. In private subnets, all three VPC endpoints must be created.

## Guardrail 3: Session Manager Requires ssmmessages Endpoint Specifically
Session Manager uses the ssmmessages endpoint, NOT the ssm endpoint. If only the ssm VPC endpoint is configured, Run Command may work but Session Manager will fail. The Session Manager plugin must also be installed on the local machine for CLI-based sessions. Session logging to S3/CloudWatch requires additional IAM permissions.

## Guardrail 4: Run Command Default Timeout Is 3600 Seconds, Not Unlimited
Run Command has a default execution timeout of 3600 seconds (1 hour). This can be configured up to 172800 seconds (48 hours) per command. However, if a command produces no output for 2 hours, it is terminated regardless of the timeout setting. Long-running scripts must write periodic output to avoid this idle timeout.

## Guardrail 5: Parameter Store Standard Tier Is Free with 4KB Limit
Standard parameters are free (up to 10,000 per account per region) with a 4KB maximum size. Advanced parameters cost $0.05/month each, support 8KB size, and enable parameter policies (expiration notifications, no-change notifications). You CANNOT convert an advanced parameter back to standard. Standard throughput is 40 TPS; higher throughput (1,000 TPS) requires opt-in and costs extra.

## Guardrail 6: Patch Baselines Are OS-Specific and Cannot Be Shared Across OS Types
Each patch baseline targets exactly one operating system (Windows, Amazon Linux 2, Ubuntu, etc.). A Windows patch baseline cannot be applied to Linux instances and vice versa. AWS provides one default baseline per OS. Custom baselines allow control over patch classification, severity, and auto-approval delays. Patch groups map instances to baselines.

## Guardrail 7: Automation Documents Use Schema Version 0.3, Command Documents Use 2.2
SSM Automation documents (runbooks) must use schemaVersion "0.3" and support steps like aws:executeAwsApi, aws:approve, aws:branch. Command documents use schemaVersion "2.2" and run directly on instances. Using the wrong schema version causes document creation to fail. Automation documents run in the SSM service context, not on instances directly.

## Guardrail 8: State Manager Associations Apply on Schedule AND on Instance Launch
State Manager associations run on the configured schedule (cron/rate) AND automatically when a new instance is launched that matches the target. This means associations can run more frequently than the schedule suggests. The association status reflects the LAST execution result, not the overall health.

## Guardrail 9: Maintenance Window Cutoff Prevents Late Task Starts
Maintenance windows have a duration and a cutoff period. Tasks that have not started by (duration - cutoff) hours will not be initiated. For example, a 4-hour window with 1-hour cutoff means tasks must start within the first 3 hours. Tasks already running when the cutoff is reached will continue. The cutoff prevents tasks from starting too close to the window end.

## Guardrail 10: Hybrid Activations Require Outbound Internet AND TLS 1.2
On-premises servers registered via hybrid activation need outbound HTTPS access to SSM endpoints (same three as EC2 instances). They also require TLS 1.2 support. The activation code and ID are one-time use per registration. Hybrid instances get an "mi-" prefix instead of "i-". The IAM service role for hybrid instances is separate from EC2 instance profiles.

## Guardrail 11: Inventory Collection Has Minimum Intervals Per Data Type
Inventory collection intervals vary by type: minimum 30 minutes for most types, but some custom inventory types can be collected more frequently. Collecting too frequently increases SSM API calls and may cause throttling. The default collection interval is 30 minutes. Inventory data is stored in SSM Inventory and can be synced to S3 via Resource Data Sync.

## Guardrail 12: SSM Document Versions Must Be Explicitly Managed
SSM documents support versioning. The "default" version is used when no version is specified. Updating a document creates a new version but does NOT automatically update the default. Associations and commands reference either a specific version, "$DEFAULT", or "$LATEST". Using "$LATEST" in production is risky as document updates immediately affect all executions.
