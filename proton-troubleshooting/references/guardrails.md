# AWS Proton Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS Proton issue.

## Guardrail 1: Template Versions Must Be Published Before Deployment
Proton template versions go through DRAFT → PUBLISHED lifecycle. Only PUBLISHED versions can be used to create or update environments and services. A REGISTRATION_FAILED version indicates a problem with the template bundle. Never suggest deploying a DRAFT or failed version.

## Guardrail 2: CloudFormation vs Terraform Provisioning Are Fundamentally Different
AWS-managed provisioning (CloudFormation) is executed by Proton directly. Self-managed provisioning (Terraform) requires external infrastructure — Proton sends provisioning requests but does NOT execute Terraform. Never suggest Proton runs terraform apply or manages Terraform state.

## Guardrail 3: Self-Managed Provisioning Requires External Infrastructure
For Terraform-based templates, you must set up your own provisioning pipeline (CodeBuild, GitHub Actions, etc.) that listens for Proton provisioning events and executes Terraform. Proton only manages the desired state — actual provisioning is your responsibility.

## Guardrail 4: Service Pipelines Are Optional and Template-Defined
Not all service templates include CI/CD pipelines. Pipeline configuration is part of the service template definition. If a service has no pipeline, it's by design — not a bug. Check the service template to determine if a pipeline is expected.

## Guardrail 5: Components Are Customer-Managed Infrastructure
Proton components allow attaching customer-managed infrastructure to service instances. Components are provisioned and managed by the customer, not by Proton. Proton tracks component state but does not auto-provision or auto-remediate components.

## Guardrail 6: Template Sync Requires Active CodeStar Connection
Template sync connects a Git repository to Proton templates for automatic version creation. This requires a CodeStar connection in AVAILABLE status. PENDING connections need manual authorization in the AWS Console. Template sync failures are often caused by connection issues.

## Guardrail 7: IAM Roles Are Separated by Function
Proton uses distinct IAM roles: the Proton service role (for Proton API operations), pipeline service role (for CI/CD pipeline execution), component role (for component provisioning), and environment provisioning role. Each has different permission requirements. Do not conflate these roles.

## Guardrail 8: Environment Outputs Feed Service Template Inputs
Environment templates produce outputs that are consumed as inputs by service templates. If environment outputs change (e.g., VPC ID, subnet IDs), dependent services may need redeployment. Always check environment outputs when service deployments fail with input errors.

## Guardrail 9: Template Bundles Must Follow Proton Schema
Template bundles require specific directory structure: schema/ (with schema.yaml), infrastructure/ (with CloudFormation/Terraform templates), and optionally pipeline/ for service templates. Missing manifests, invalid schemas, or malformed templates cause registration failures.

## Guardrail 10: Drift Detection Does Not Auto-Remediate
Proton can detect drift between desired and actual infrastructure state, but it does NOT automatically fix drift. Manual intervention (update or redeploy) is required to resolve drift. Drift detection is informational only.

## Guardrail 11: CodeStar Connections Require Manual Console Authorization
CodeStar connections created via API or CLI start in PENDING status. They must be manually authorized through the AWS Console by completing the OAuth flow with the Git provider (GitHub, Bitbucket, etc.). This cannot be automated via API.

## Guardrail 12: Deployment Ordering Matters — Environments Before Services
Environments must be fully provisioned before services can be deployed into them. Service templates reference environment outputs. Attempting to create a service before its environment is ready will fail. Always verify environment deployment status before troubleshooting service failures.
