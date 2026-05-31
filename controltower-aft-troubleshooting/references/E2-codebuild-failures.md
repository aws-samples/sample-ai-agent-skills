---
title: "E2 — CodeBuild Failures"
description: "Diagnose and resolve CodeBuild project failures in AFT pipelines"
status: active
severity: HIGH
triggers: ["CodeBuild failed", "build error", "build timeout", "Terraform execution failed in CodeBuild"]
owner: devops-agent
objective: "Resolve CodeBuild failures that block AFT Terraform execution"
context: "AFT uses CodeBuild to execute Terraform plan and apply operations. CodeBuild failures are the most common AFT issue and contain the actual Terraform error messages. Build logs are the primary diagnostic tool."
---

## Phase 1 — Triage

MUST:
- List recent builds: `aws codebuild list-builds-for-project --project-name <project-name> --max-items 5`
- Get build details and logs: `aws codebuild batch-get-builds --ids <build-id> --query 'builds[0].{Status:buildStatus,Phase:currentPhase,StartTime:startTime,EndTime:endTime}'`
- Review build logs in CloudWatch: `aws logs get-log-events --log-group-name /aws/codebuild/<project-name> --log-stream-name <build-id>`

SHOULD:
- Check CodeBuild project configuration: `aws codebuild batch-get-projects --names <project-name>`
- Verify CodeBuild service role permissions
- Check for build timeout issues (default vs configured timeout)

MAY:
- Check CodeBuild compute type and image for compatibility
- Review buildspec.yml for configuration issues
- Check VPC configuration if CodeBuild runs in a VPC

## Phase 2 — Remediate

MUST:
- Fix the Terraform error identified in build logs
- Ensure CodeBuild service role has permissions for Terraform operations
- Fix network connectivity if builds timeout during provider download

SHOULD:
- Increase build timeout if Terraform operations take longer than default
- Verify CodeBuild image has required tools (Terraform, AWS CLI, Python)
- Check environment variables are correctly set in the CodeBuild project

MAY:
- Configure CodeBuild caching for Terraform providers to speed up builds
- Set up build notifications for failure alerting

## Safety Ratings

```
safety_ratings:
  - "codebuild list-builds-for-project: GREEN — read-only build listing"
  - "codebuild batch-get-builds: GREEN — read-only build details"
  - "logs get-log-events: GREEN — read-only log access"
  - "codebuild batch-get-projects: GREEN — read-only project configuration"
```

## Escalation Conditions

- CodeBuild consistently times out despite timeout increases
- Build environment lacks required tools and image cannot be changed
- VPC configuration prevents CodeBuild from accessing required resources
- CodeBuild service role permissions are restricted by SCP

## Data Sensitivity

- CodeBuild logs: HIGH — contain Terraform output, resource details, and potential error messages with sensitive data
- CodeBuild project configuration: MEDIUM — reveals build environment and role configuration
- Environment variables: HIGH — may contain secrets or configuration values
- Build artifacts: HIGH — may contain Terraform plans with resource details

## Prohibited Actions

- NEVER expose CodeBuild environment variables containing secrets in logs
- NEVER modify AFT CodeBuild projects outside of Terraform
- NEVER grant CodeBuild service role AdministratorAccess
- NEVER disable build logging — logs are essential for troubleshooting

## Phase 3 — Rollback

1. If CodeBuild project configuration was modified, restore via AFT Terraform
2. If build artifacts contain sensitive data, ensure S3 bucket has appropriate access controls
3. If environment variables were changed, revert to previous values
4. Document build failures and resolutions
5. Verify the pipeline can execute successfully after fixes

## Common Issues

- symptoms: "Build fails with 'Error: Failed to install provider'"
  diagnosis: "CodeBuild cannot reach the Terraform registry. VPC configuration blocks internet access."
  resolution: "If CodeBuild is in VPC, ensure NAT gateway is configured. Check security group outbound rules."

- symptoms: "Build times out at PROVISIONING phase"
  diagnosis: "CodeBuild cannot provision compute resources, often due to VPC subnet capacity."
  resolution: "Check VPC subnet available IP addresses. Try a different subnet or increase subnet size."

## Output Format

```yaml
root_cause: "codebuild — <specific_cause>"
evidence:
  - type: build_error
    content: "<build status, phase, and log excerpts>"
severity: HIGH
mitigation:
  immediate: "Fix build failure cause"
  long_term: "Implement build caching and monitoring"
```

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
