# AWS Control Tower AFT Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AFT issue.

## Guardrail 1: AFT Has a Multi-Pipeline Architecture
AFT uses separate CodePipeline pipelines for account-request, account-provisioning, global-customizations, and account-customizations. Each pipeline can fail independently. Identify which pipeline is failing before diagnosing — do not assume all AFT failures are in the same pipeline.

## Guardrail 2: DynamoDB Is the Source of Truth
The `aft-request` DynamoDB table tracks account request status. Pipeline status alone does not reflect the true state of account provisioning. Always check DynamoDB for the authoritative account request status before concluding on provisioning state.

## Guardrail 3: Terraform State Must Never Be Manually Deleted
AFT stores Terraform state in S3 with DynamoDB locking. Deleting or corrupting state files causes orphaned resources and irrecoverable drift. Never suggest deleting state files. Use `terraform state` commands for state manipulation, and only with explicit approval.

## Guardrail 4: Global Customizations Run Before Account Customizations
AFT executes global customizations first, then account-specific customizations. A failure in global customizations blocks all account customizations. Diagnose global customization failures before investigating account-specific ones.

## Guardrail 5: AFT Does Not Create SSO Permission Sets
AFT assigns existing SSO permission sets to accounts. It does NOT create permission sets. If a permission set referenced in the account request does not exist, the provisioning fails. Verify permission set existence before blaming AFT.

## Guardrail 6: Account Provisioning and Customization Are Separate
Account provisioning (creating the AWS account via Control Tower) and account customization (applying Terraform configurations) are separate processes. Provisioning failures are Control Tower issues; customization failures are Terraform/CodeBuild issues.

## Guardrail 7: AFT Runs in the AFT Management Account
AFT infrastructure (pipelines, Lambda functions, DynamoDB tables) runs in the AFT management account, not the Control Tower management account. Ensure you are investigating in the correct account.

## Guardrail 8: CodeBuild Logs Are the Primary Diagnostic
Most AFT failures manifest as CodeBuild failures. The CodeBuild build logs contain the actual error messages, Terraform output, and stack traces. Always check CodeBuild logs before diagnosing at a higher level.

## Guardrail 9: Terraform Provider Versions Must Be Compatible
AFT uses specific Terraform provider versions (AWS, CT). Mismatched provider versions cause plan/apply failures. Check `versions.tf` and provider lock files before assuming code issues.

## Guardrail 10: Account Request Schema Is Strict
The account request input (in the `aft-account-request` repository) must follow the exact schema. Missing required fields, incorrect types, or invalid values cause silent failures in the account-request pipeline.

## Guardrail 11: VPC Configuration Affects CodeBuild Connectivity
If AFT CodeBuild projects are configured to run in a VPC, they need NAT gateway access for internet connectivity (pulling Terraform providers, accessing AWS APIs). VPC misconfiguration causes CodeBuild timeouts, not explicit errors.

## Guardrail 12: AFT Upgrade Requires Version Compatibility Matrix
Upgrading AFT requires checking compatibility between the AFT module version, Terraform version, AWS provider version, and Control Tower version. Skipping versions or mixing incompatible versions causes deployment failures.
