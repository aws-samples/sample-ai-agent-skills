---
title: "C1 — Build Upload Errors"
description: "Diagnose GameLift build upload and validation errors"
status: active
severity: HIGH
triggers: ["build upload failed", "build not ready", "upload-build error", "build validation failed"]
owner: devops-agent
objective: "Resolve build upload, validation, and readiness issues"
context: "Builds contain compiled game server executables. They must be uploaded to GameLift and reach READY status before fleet creation. Builds are immutable once uploaded."
---

## Phase 1 — Triage

MUST:
- Check build status: `aws gamelift describe-build --build-id <build-id>`
- List builds: `aws gamelift list-builds --status FAILED`
- Verify build files exist and are valid: check local build directory

SHOULD:
- Check S3 upload permissions if using S3-based upload
- Verify build operating system matches fleet configuration
- Check build size against limits

MAY:
- Review CloudTrail for CreateBuild errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateBuild --max-results 10`
- Verify the executable has correct permissions (chmod +x on Linux)

## Phase 2 — Remediate

MUST:
- Ensure build executable has execute permissions: `chmod +x <server-binary>`
- Upload build: `aws gamelift upload-build --name <name> --build-version <version> --build-root <path> --operating-system AMAZON_LINUX_2`
- Wait for build to reach READY status before creating fleet

SHOULD:
- Include all required shared libraries and dependencies in the build
- Test the server binary locally before uploading
- Use install.sh script for additional setup during fleet activation

MAY:
- Use S3 for large builds: `aws gamelift create-build --name <name> --storage-location Bucket=<bucket>,Key=<key>,RoleArn=<role-arn>`
- Automate build upload in CI/CD pipeline

## Safety Ratings

```
safety_ratings:
  - "describe-build / list-builds: GREEN — read-only inspection"
  - "upload-build: YELLOW — uploads server binary to GameLift"
  - "create-build (S3): YELLOW — creates build from S3 object"
  - "delete-build: RED — removes build permanently (fleets using it are unaffected)"
```

## Escalation Conditions

- Build upload consistently fails with network errors
- Build validation fails with no clear error message
- Build size exceeds GameLift limits
- S3 cross-account access required for build upload

## Data Sensitivity

- `aws gamelift describe-build`: LOW — build metadata
- Build binary: HIGH — contains proprietary game server code
- S3 build storage: HIGH — contains server executables
- `aws cloudtrail lookup-events`: MEDIUM — upload activity records

## Prohibited Actions

- NEVER upload builds containing hardcoded credentials or secrets
- NEVER delete builds that are referenced by active fleets
- NEVER upload untested builds to production
- NEVER share build S3 buckets publicly

## Phase 3 — Rollback

1. If wrong build was uploaded: create a new build with correct files (old build remains)
2. If build was deleted: re-upload from source
3. If fleet was created with wrong build: delete fleet and create with correct build
4. If S3 permissions were modified: restore original bucket policy
5. Document build versioning and update deployment procedures

## Common Issues

- symptoms: "Build status stays in INITIALIZED and never reaches READY"
  diagnosis: "Build upload did not complete successfully, or the uploaded files are invalid."
  resolution: "Re-upload the build. Ensure the server executable has execute permissions. Verify all dependencies are included."

## Output Format

```yaml
root_cause: "build_upload — <specific_cause>"
evidence:
  - type: build_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix build upload issue"
  long_term: "Implement CI/CD pipeline for automated build validation and upload"
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
