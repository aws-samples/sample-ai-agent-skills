# AWS Resilience Hub Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Resilience Hub issue.

## Guardrail 1: Resource Discovery Requires Explicit Sources
Resilience Hub discovers resources through CloudFormation stacks, Resource Groups, AppRegistry, or Terraform state files. Resources not managed by these sources are invisible to Resilience Hub. Manually created resources must be added to a Resource Group or CFN stack first.

## Guardrail 2: Assessments Are Read-Only
Assessments evaluate architecture against resiliency policies but do NOT make any changes to infrastructure. They report compliance status and provide recommendations. Implementing recommendations is a manual process. Never claim assessments auto-remediate.

## Guardrail 3: Policy Must Be Attached Before Assessment
An application must have a resiliency policy attached before running an assessment. Without a policy, there are no RTO/RPO targets to evaluate against. Verify policy attachment: aws resiliencehub describe-app --app-arn <arn> --query 'app.policyArn'.

## Guardrail 4: RTO/RPO Targets Are Per Disruption Type
Resiliency policies define separate RTO/RPO targets for four disruption types: Software, Hardware, AZ, and Region. A component may be compliant for AZ disruption but non-compliant for Region disruption. Always specify which disruption type when discussing compliance.

## Guardrail 5: Assessment Duration Varies
Assessment duration depends on application complexity and number of resources. Large applications with hundreds of resources can take 10+ minutes. Do not assume assessment failure if it's still in INPROGRESS state. Check assessmentStatus field.

## Guardrail 6: Drift Detection Compares Against Last Assessment
Drift detection compares current infrastructure against the last successful assessment, not against the resiliency policy directly. If no assessment has been run, drift detection has no baseline. Run an assessment first.

## Guardrail 7: SOP Documents Are Templates
Generated SOPs (SSM documents) are templates based on best practices. They require customization for your specific environment — account IDs, resource names, region-specific configurations. Do not execute generated SOPs without review and customization.

## Guardrail 8: Component Grouping Affects Compliance
Resilience Hub groups resources into application components. The grouping affects how compliance is evaluated. Incorrect grouping (e.g., putting independent services in one component) can produce misleading compliance results.

## Guardrail 9: Cross-Region Assessment Requires Multi-Region Setup
To assess Region-level resilience, the application must have resources in multiple regions. Resilience Hub cannot assess Region-level resilience for single-region applications. It will report non-compliant for Region disruption type.

## Guardrail 10: Terraform State File Must Be Accessible
When using Terraform as a resource source, Resilience Hub needs access to the Terraform state file in S3. IAM permissions must allow reading the state file. State file format must be compatible with the supported Terraform version.

## Guardrail 11: Application Versions Track Changes
Resilience Hub maintains application versions. Publishing a new version creates a snapshot. Assessments run against published versions. Draft changes must be published before they're included in assessments.

## Guardrail 12: Recommendations May Not Match Your Architecture Decisions
Recommendations are based on AWS best practices for resilience. They may suggest changes (e.g., Multi-AZ, cross-region replication) that conflict with your cost constraints or architecture decisions. Recommendations are suggestions, not requirements.
