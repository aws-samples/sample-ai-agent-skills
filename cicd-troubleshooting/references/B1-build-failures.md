---
title: "B1 — Build Failures"
description: "Diagnose and resolve CodeBuild build failures"
status: active
severity: HIGH
triggers:
  - "build failed"
  - "build error"
  - "FAILED"
  - "phase failed"
  - "buildspec"
owner: devops-agent
objective: "Identify why the CodeBuild build failed and resolve the issue"
context: "CodeBuild builds go through phases: SUBMITTED, QUEUED, PROVISIONING, DOWNLOAD_SOURCE, INSTALL, PRE_BUILD, BUILD, POST_BUILD, UPLOAD_ARTIFACTS, FINALIZING. Failures can occur at any phase. The build logs and phase details provide the primary diagnostic information."
---

## Phase 1 — Triage

MUST:
- Get build details and phase information: `aws codebuild batch-get-builds --ids <build-id>`
- Identify which phase failed: check `phases` array in build details for `phaseStatus: FAILED`
- Check build logs in CloudWatch: `aws logs get-log-events --log-group-name /aws/codebuild/<project-name> --log-stream-name <build-id>`
- Verify buildspec.yml exists and is valid: check `source.buildspec` in project configuration

SHOULD:
- Check project configuration: `aws codebuild batch-get-projects --names <project-name>`
- Verify the CodeBuild service role has required permissions: `aws iam list-attached-role-policies --role-name <codebuild-role>`
- Check environment variables for missing or incorrect values
- Verify the build image exists and is accessible

MAY:
- Check if the build was stopped manually: look for `buildStatus: STOPPED`
- Review recent project configuration changes in CloudTrail
- Check CodeBuild service quotas: `aws service-quotas get-service-quota --service-code codebuild --quota-code L-2DC20C30`

## Phase 2 — Remediate

MUST:
- If DOWNLOAD_SOURCE failed: verify source repository access, credentials, and branch/commit
- If INSTALL failed: check buildspec install phase commands, runtime versions, and package availability
- If BUILD failed: review build commands in buildspec, check for compilation or test errors in logs
- If UPLOAD_ARTIFACTS failed: verify artifact configuration and S3 permissions

SHOULD:
- Fix buildspec.yml syntax errors (YAML validation)
- Update environment variables if referencing missing secrets or parameters
- Ensure the build image has required tools and runtimes

MAY:
- Run the build locally using `codebuild-local` agent for faster debugging
- Switch to a different managed build image if the current one lacks required tools

## Common Issues

- symptoms: "DOWNLOAD_SOURCE phase failed — unable to access source"
  diagnosis: "CodeBuild cannot access the source repository. Credentials or permissions issue."
  resolution: "Verify source credentials. For CodeCommit: check service role has codecommit:GitPull. For GitHub: check OAuth token or CodeStar Connection."

- symptoms: "INSTALL phase failed — runtime version not found"
  diagnosis: "Requested runtime version is not available in the build image."
  resolution: "Check available runtimes for the image. Use `runtime-versions` in buildspec install phase with a supported version."

- symptoms: "BUILD phase failed — command exit code non-zero"
  diagnosis: "A build command returned a non-zero exit code (compilation error, test failure, etc.)."
  resolution: "Check build logs for the specific error. Fix the code or test, then retry."

- symptoms: "Build fails with 'buildspec.yml not found'"
  diagnosis: "The buildspec file is not in the expected location or has a different name."
  resolution: "Ensure buildspec.yml is in the repository root, or specify the path in the project source configuration."

## Output Format

```yaml
root_cause: "Build failure — <specific_phase> — <specific_cause>"
evidence:
  - type: build_phases
    content: "<failed phase details>"
  - type: build_logs
    content: "<relevant log entries>"
severity: HIGH
mitigation:
  immediate: "Fix the identified issue and retry the build"
  long_term: "Add build validation in pre-commit hooks, use consistent build images"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| If DOWNLOAD_SOURCE failed: verify source repository access and credentials | GREEN |
| If INSTALL failed: check buildspec install phase commands and runtime versions | GREEN |
| If BUILD failed: review build commands, check for compilation or test errors | GREEN |
| If UPLOAD_ARTIFACTS failed: verify artifact configuration and S3 permissions | YELLOW |
| Fix buildspec.yml syntax errors | GREEN |
| Update environment variables if referencing missing secrets | YELLOW |
| Ensure the build image has required tools and runtimes | GREEN |
| Run the build locally using codebuild-local agent | GREEN |
| Switch to a different managed build image | GREEN |

## Escalation Conditions

- Fix requires modifying production pipeline
- Fix requires changing deployment strategy

## Data Sensitivity

- **HIGH**: build logs (may contain secrets), artifact contents
- Build logs may contain environment variables, credentials, and source code snippets

## Prohibited Actions

- NEVER suggest storing secrets in buildspec.yml
- NEVER suggest skipping approval stages in production pipelines

## Phase 3 — Rollback

1. If buildspec.yml was modified, revert to the previous version in source control
2. If environment variables were changed, restore previous values in the CodeBuild project
3. If the build image was changed, restore the previous image: `aws codebuild update-project --name <project> --environment image=<previous-image>`
4. If S3 artifact permissions were modified, revert the IAM policy
5. Verify rollback by triggering a new build and confirming it uses the previous configuration

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
