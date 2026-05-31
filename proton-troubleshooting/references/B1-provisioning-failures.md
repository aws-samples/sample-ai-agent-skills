---
title: "B1 — Proton Environment Provisioning Failures"
description: "Diagnose environment provisioning and deployment failures"
status: active
severity: HIGH
triggers:
  - "environment provisioning failed"
  - "DEPLOYMENT_FAILED"
  - "environment creation error"
  - "CloudFormation stack failed"
owner: devops-agent
objective: "Identify why a Proton environment fails to provision and resolve the deployment issue"
context: "Environment provisioning creates shared infrastructure using CloudFormation or Terraform. Failures stem from template errors, IAM permission issues, resource limits, or VPC/networking problems."
---

## Phase 1 — Triage

MUST:
- Check environment status: `aws proton get-environment --name <env-name> --query '{Status:deploymentStatus,Message:deploymentStatusMessage}'`
- Check environment outputs: `aws proton list-environment-outputs --environment-name <env-name>`
- For CloudFormation: check stack events: `aws cloudformation describe-stack-events --stack-name <proton-stack-name> --query 'StackEvents[?ResourceStatus==\`CREATE_FAILED\`]'`
- Check CloudTrail for provisioning events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateEnvironment --max-results 10`
- Verify Proton service role permissions: `aws iam get-role --role-name <proton-service-role>`

SHOULD:
- Check if resource limits are exceeded (VPCs, EIPs, etc.)
- Verify template inputs match expected values
- Check for resource naming conflicts

MAY:
- For self-managed provisioning, check the external provisioning pipeline
- Review CloudFormation drift if updating an existing environment

## Phase 2 — Remediate

MUST:
- Fix template errors identified in stack events or status message
- Grant missing IAM permissions to the Proton service role
- Request service quota increases if resource limits are hit

SHOULD:
- Validate template inputs before deployment
- Test templates in a staging account first
- Use unique naming to avoid resource conflicts

MAY:
- Delete failed environment and recreate after fixing issues
- Use CloudFormation change sets to preview changes before applying

## Common Issues

- symptoms: "Environment DEPLOYMENT_FAILED with CloudFormation error"
  diagnosis: "CloudFormation template has resource creation errors."
  resolution: "Check CloudFormation stack events for the specific resource failure."

- symptoms: "Environment provisioning times out"
  diagnosis: "Resource creation is slow or stuck (e.g., RDS, EKS cluster)."
  resolution: "Check individual resource status. Some resources take 15-30 minutes."

- symptoms: "Permission denied during provisioning"
  diagnosis: "Proton service role lacks permissions to create the required resources."
  resolution: "Add missing permissions to the Proton service role."

## Output Format

```yaml
root_cause: "environment_provisioning_failure — <specific_cause>"
evidence:
  - type: deployment_status
    content: "<status and statusMessage>"
  - type: stack_events
    content: "<CloudFormation failure events>"
  - type: iam_role
    content: "<Proton service role permissions>"
severity: HIGH
mitigation:
  immediate: "Fix the identified template, permission, or resource issue"
  long_term: "Add pre-deployment validation and staging environment testing"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: LOW — provisioning failures do not expose data
- Service Disruption Risk: HIGH — provisioning failures prevent environment deployment
- Compliance Impact: MEDIUM — provisioning must follow approved templates

## Escalation Conditions
- Provisioning fails with CloudFormation stack errors
- Provisioning blocked by IAM or SCP policies
- Provisioning leaves resources in inconsistent state
- Provisioning timeout causes partial deployment
- Multiple provisioning operations fail simultaneously

## Data Sensitivity
- CloudFormation stack parameters — INTERNAL (may contain CONFIDENTIAL data)
- IAM role configurations — INTERNAL
- Resource configurations — INTERNAL
- Provisioning logs — INTERNAL
- Never expose provisioning parameters containing secrets

## Prohibited Actions
- Do NOT force-delete failed provisioning stacks without cleanup
- Do NOT bypass provisioning approval workflows
- Do NOT provision with overly permissive IAM roles
- Do NOT ignore provisioning failure patterns
- Do NOT provision without monitoring
- Do NOT modify provisioning parameters during active operations

## Phase 3 — Rollback
- Delete failed provisioning stacks and clean up resources
- Restore previous environment configurations
- Revert IAM role changes
- Verify resource cleanup after failed provisioning
- Document provisioning failures and root causes
- Update provisioning templates based on findings

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
