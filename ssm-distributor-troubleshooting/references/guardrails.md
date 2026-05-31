# SSM Distributor Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any SSM Distributor issue.

## Guardrail 1: Distributor Packages Are SSM Documents of Type Package
Distributor packages are a specific document type (Package) in SSM. They are NOT regular Command or Automation documents. Package documents contain a manifest with platform-specific installers, install/uninstall scripts, and version metadata. Do not confuse with other SSM document types.

## Guardrail 2: SSM Agent Must Be Running and Connected
Package operations require the SSM Agent to be running on the target instance with an active PingStatus. Instances showing "ConnectionLost" or no PingStatus cannot receive packages. Always verify instance connectivity before diagnosing installation failures.

## Guardrail 3: Package Versions Are Immutable
Once a package version is created, it cannot be modified. To fix a broken package, you must create a new version. The default version can be changed, but existing versions remain unchanged. Do not suggest editing an existing package version.

## Guardrail 4: Custom Packages Require Same-Region S3 Buckets
Custom package artifacts must be stored in an S3 bucket in the same region as the target instances. Cross-region S3 access is not supported for Distributor packages. The SSM Agent downloads artifacts directly from S3.

## Guardrail 5: Platform-Specific Installers Are Required
Each target platform (Windows, Linux, macOS) requires its own installer in the package manifest. A package without a Linux installer cannot be installed on Linux instances. The manifest.json file defines which platforms are supported.

## Guardrail 6: Associations Execute on Schedule Not Immediately
Association-based package installations follow the configured schedule (cron or rate expression). Creating an association does not immediately install the package unless ApplyOnlyAtCronInterval is false. Verify the schedule expression and last execution time.

## Guardrail 7: AWS-Managed Packages Cannot Be Modified
Third-party packages owned by Amazon (e.g., AmazonCloudWatchAgent, AWSPVDriver) are AWS-managed. You cannot modify their content, scripts, or manifest. You can only choose which version to install. Do not suggest editing AWS-managed packages.

## Guardrail 8: Installation Requires Sufficient Disk Space
Package installation requires sufficient disk space on the target instance for downloading and extracting the package. Large packages (e.g., agents, runtimes) may fail silently if disk space is insufficient. Check available disk space before concluding on installation failures.

## Guardrail 9: Uninstallation Scripts Must Be Idempotent
Uninstall scripts should handle cases where the software is already removed or partially installed. Non-idempotent uninstall scripts cause repeated failures. Verify the uninstall script handles edge cases.

## Guardrail 10: S3 Bucket Policies Affect Package Access
The S3 bucket storing custom packages must allow the SSM Agent's IAM role to perform s3:GetObject. Bucket policies, VPC endpoints, and encryption settings can all block access. Check all layers of S3 access control.

## Guardrail 11: Command Output Is Truncated at 48000 Characters
SSM command output (StandardOutputContent and StandardErrorContent) is truncated at 48,000 characters. For verbose installers, critical error messages may be truncated. Use S3 or CloudWatch Logs output for full logs.

## Guardrail 12: Distributor Is Regional
Distributor packages are regional resources. A package created in us-east-1 is not available in eu-west-1. For multi-region deployments, packages must be created or shared in each target region.
