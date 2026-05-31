# Landing Zone Accelerator Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any LZA issue.

## Guardrail 1: Single Pipeline with Cascading Stages
LZA uses a single CodePipeline (AWSAccelerator-Pipeline) with multiple stages that deploy to different accounts and regions. A failure in any stage blocks all subsequent stages. Identify the exact failing stage before diagnosing — do not assume the last stage is the problem.

## Guardrail 2: Configuration Files Are the Primary Input
LZA reads YAML configuration files (accounts-config.yaml, organization-config.yaml, network-config.yaml, security-config.yaml, global-config.yaml, customizations-config.yaml) from an S3 bucket or CodeCommit repository. Most LZA failures trace back to configuration file errors. Always validate configuration files first.

## Guardrail 3: CDK Synth and CloudFormation Deploy Are Different Phases
CDK synthesis (cdk synth) parses configuration files and generates CloudFormation templates. CloudFormation deployment applies those templates. Synth failures are configuration/code errors; deployment failures are resource-level errors. Diagnose at the correct phase.

## Guardrail 4: Multi-Account Multi-Region Deployment
LZA deploys CloudFormation stacks across multiple accounts and regions simultaneously. A single configuration error can cause failures in dozens of stacks. Always identify the specific account and region where the failure occurs.

## Guardrail 5: Changes Outside LZA Cause Drift
LZA manages AWS Organizations, SCPs, VPCs, security groups, IAM roles, and many other resources. Manual changes to these resources outside of LZA cause drift that can break subsequent pipeline runs. Always check for drift before diagnosing deployment failures.

## Guardrail 6: Version Upgrades Must Be Sequential
LZA upgrades must follow the documented version path. Skipping versions can cause state inconsistencies, schema changes, and deployment failures. Always check the current version and the upgrade path before attempting an upgrade.

## Guardrail 7: Organization Config Controls Account Structure
The organization-config.yaml controls OU structure, SCPs, and account placement. Changes to this file can move accounts between OUs, apply or remove SCPs, and affect the entire organization. Review changes carefully before committing.

## Guardrail 8: Network Config Is the Most Complex
The network-config.yaml controls VPCs, subnets, transit gateways, route tables, security groups, NACLs, and more. It is the most complex configuration file and the most common source of deployment failures. Validate network configuration thoroughly.

## Guardrail 9: Security Config Affects All Accounts
The security-config.yaml controls GuardDuty, Security Hub, Macie, Config rules, CloudTrail, and other security services. Changes affect all accounts in the organization. Security configuration errors can disable security monitoring.

## Guardrail 10: CloudFormation Stack Names Follow a Pattern
LZA CloudFormation stacks follow the naming pattern `AWSAccelerator-<Phase>-<Account>-<Region>`. Use this pattern to identify which account and region a stack belongs to. Stack names are the primary identifier for troubleshooting.

## Guardrail 11: Pipeline Runs Are Not Idempotent for All Changes
While most LZA operations are idempotent, some changes (account creation, OU restructuring) are not. Re-running the pipeline after a failure may not fix the issue if the underlying configuration is wrong. Fix the configuration before retrying.

## Guardrail 12: Customizations Config Extends but Does Not Override
The customizations-config.yaml adds resources on top of LZA-managed infrastructure. It does NOT override LZA-managed resources. Attempting to manage the same resource in both LZA core config and customizations causes conflicts.
