# AWS Service Catalog Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Service Catalog issue.

## Guardrail 1: Admin vs End User Permissions Are Separate
Service Catalog has two distinct permission models. Administrators manage portfolios, products, and constraints. End users provision and manage provisioned products. Admin APIs (e.g., search-products-as-admin) require different permissions than end user APIs (e.g., search-products).

## Guardrail 2: Launch Constraints Define the Provisioning Role
Launch constraints specify the IAM role used to provision the CloudFormation stack. Without a launch constraint, the end user's own IAM permissions are used. Most provisioning failures occur because no launch constraint is set and the end user lacks CloudFormation permissions.

## Guardrail 3: Template Constraints Use JSON Rules Syntax
Template constraints restrict CloudFormation parameter values using JSON rules syntax, NOT IAM policy syntax. The rules format uses "Rule" with "Assertions" containing "Assert" and "AssertDescription". Incorrect syntax silently fails or produces confusing errors.

## Guardrail 4: Provisioned Products Are CloudFormation Stacks
Every provisioned product creates a CloudFormation stack. Provisioning failures are almost always CloudFormation stack failures. Always check the underlying stack events for the actual error: aws cloudformation describe-stack-events --stack-name <stack>.

## Guardrail 5: Portfolio Sharing Has Multiple Methods
Portfolios can be shared via AWS account ID, AWS Organizations, or organizational unit. Organization sharing requires enabling Service Catalog integration with Organizations. Account-level sharing and organization sharing have different permission requirements.

## Guardrail 6: Product Versions Are Immutable
Once a product version (provisioning artifact) is created, its template cannot be modified. To update a template, create a new product version. Old versions can be disabled but not deleted if provisioned products reference them.

## Guardrail 7: Tag Options Are Not Resource Tags
Tag options are a Service Catalog governance feature that enforces specific tag key-value pairs during provisioning. They are different from CloudFormation resource tags. Tag options are associated with portfolios or products and presented to users during provisioning.

## Guardrail 8: StackSet Constraints Enable Multi-Account Deployment
StackSet constraints allow provisioned products to deploy resources across multiple accounts and regions using CloudFormation StackSets. They require specific IAM roles (AWSCloudFormationStackSetAdministrationRole and AWSCloudFormationStackSetExecutionRole).

## Guardrail 9: Terraform Products Require External Engine
Terraform products use the Terraform Cloud or Terraform Enterprise engine, not a local Terraform installation. They require a Terraform Cloud workspace connection and proper API token configuration. Terraform open-source is not supported.

## Guardrail 10: Self-Service Actions Extend Provisioned Products
Self-service actions allow end users to perform operational tasks (like restart, backup) on provisioned products. They use AWS Systems Manager documents. Self-service actions are not the same as product updates.

## Guardrail 11: Imported Portfolios Cannot Be Modified
When a portfolio is shared to another account, the recipient gets an imported portfolio. Imported portfolios cannot be modified — products, constraints, and tag options are read-only. The source account must make changes.

## Guardrail 12: Record Details Track All Provisioning Operations
Every provisioning operation (provision, update, terminate) creates a record. Records contain status, errors, and output details. Always check the record for detailed error information: aws servicecatalog describe-record --id <record-id>.
