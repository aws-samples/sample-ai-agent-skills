# Amazon CodeCatalyst Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any CodeCatalyst issue.

## Guardrail 1: Builder ID Is Not IAM
CodeCatalyst uses AWS Builder ID for authentication, which is a separate identity system from AWS IAM. Users log in with Builder ID credentials, not IAM user credentials. Do not suggest IAM user login, access keys, or IAM console access for CodeCatalyst authentication.

## Guardrail 2: Spaces Are Not AWS Accounts
A CodeCatalyst space is an organizational container for projects and members. It is NOT an AWS account. A space can be connected to multiple AWS accounts via AWS connections. Do not confuse space configuration with AWS account settings.

## Guardrail 3: Workflows Are Not GitHub Actions
CodeCatalyst workflows use YAML definitions that are similar to but distinct from GitHub Actions. Syntax, action names, environment variables, and runner configuration differ. Do not copy GitHub Actions workflows directly into CodeCatalyst.

## Guardrail 4: Dev Environments Are Not EC2 Instances
CodeCatalyst dev environments are managed cloud compute instances. They are NOT EC2 instances. You cannot SSH into them directly, attach EBS volumes, or manage them with EC2 APIs. Use the CodeCatalyst API or IDE integration.

## Guardrail 5: AWS Connections Require IAM Roles
Connecting a CodeCatalyst space to an AWS account requires creating an IAM role with a trust policy that allows the CodeCatalyst service principal. The role must exist in the target AWS account. Do not confuse this with cross-account IAM roles.

## Guardrail 6: Blueprints Are One-Time Generators
Blueprints generate initial project structure and configuration. After project creation, the blueprint does NOT continuously manage or update the project. Changes to the blueprint do not automatically propagate to existing projects.

## Guardrail 7: Source Repositories Have Two Types
CodeCatalyst supports native source repositories (managed by CodeCatalyst) and linked repositories (GitHub, Bitbucket). They have different capabilities and limitations. Do not assume all repository features work identically across types.

## Guardrail 8: Workflow Actions Run in Managed Compute
Workflow actions run in CodeCatalyst-managed compute environments. You do not manage the underlying infrastructure. Action compute has specific resource limits (CPU, memory, storage) that cannot be modified like EC2 instance types.

## Guardrail 9: Environment Configuration Is Project-Scoped
CodeCatalyst environments are project-scoped deployment targets. They are configured within a project and linked to AWS accounts via connections. Do not confuse CodeCatalyst environments with AWS environments (like Elastic Beanstalk environments).

## Guardrail 10: Pull Requests Use CodeCatalyst-Native Review
CodeCatalyst has its own pull request and code review system. While similar to GitHub PRs, the review workflow, approval rules, and merge strategies may differ. Do not assume GitHub PR features are available.

## Guardrail 11: Space Billing Is Separate from AWS Billing
CodeCatalyst has its own billing model (Free and Enterprise tiers) separate from AWS account billing. Dev environment compute and workflow minutes are billed through CodeCatalyst, not through the connected AWS account.

## Guardrail 12: IAM Roles for Workflows Need Specific Trust Policy
IAM roles used by CodeCatalyst workflows must have a trust policy that allows the CodeCatalyst service principal (codecatalyst.amazonaws.com). Standard IAM roles without this trust policy cannot be assumed by workflows.
