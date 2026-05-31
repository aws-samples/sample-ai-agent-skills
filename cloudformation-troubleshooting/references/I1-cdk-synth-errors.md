---
title: "I1 — CDK Synth Errors"
description: "Diagnose and resolve CDK synthesis (template generation) failures"
status: active
severity: MEDIUM
triggers:
  - "cdk synth"
  - "synthesis failed"
  - "synth error"
  - "construct error"
  - "CDK app error"
owner: devops-agent
objective: "Fix CDK application code or configuration to produce a valid CloudFormation template"
context: "CDK synth runs locally and generates a CloudFormation template from CDK constructs. Synth errors are code-level issues — TypeScript/Python/Java compilation errors, missing context values, construct library version mismatches, or invalid construct configurations. These are NOT CloudFormation errors — the template hasn't been sent to AWS yet."
---

## Phase 1 — Triage

MUST:
- Run synth with verbose output: `cdk synth --verbose 2>&1`
- Read the error message — it's a local code error, not a CloudFormation error
- Check if it's a compilation error (TypeScript, Python, Java) or a construct validation error
- Verify CDK dependencies are installed: `npm ls aws-cdk-lib` or `pip show aws-cdk-lib`

SHOULD:
- Check for version mismatches between CDK CLI and library: `cdk --version` vs `npm ls aws-cdk-lib`
- Check if context values are missing: `cdk context`
- Verify the CDK app entry point is correct in `cdk.json`

MAY:
- Check if the error is in a custom construct or a third-party construct library
- Review recent changes to the CDK app code
- Check if environment variables or AWS credentials are required for synth

## Phase 2 — Remediate

MUST:
- Fix the code error reported by synth
- For compilation errors: fix TypeScript/Python/Java syntax
- For construct validation: fix the construct configuration (e.g., missing required props)
- For missing context: provide context values or use `cdk synth --context key=value`
- Re-run `cdk synth` to verify the fix

SHOULD:
- Keep CDK CLI and library versions in sync
- Run `npm install` or `pip install -r requirements.txt` to ensure dependencies are current
- Use `cdk diff` after synth to review the generated template

MAY:
- Use `cdk synth --json` to output JSON for easier debugging
- Review the generated template in `cdk.out/` for correctness

## Common Issues

- symptoms: "Error: Cannot find module 'aws-cdk-lib'"
  diagnosis: "CDK library not installed or wrong version."
  resolution: "Run npm install or pip install aws-cdk-lib."

- symptoms: "Error: <Construct> requires property 'X'"
  diagnosis: "A required construct property is missing."
  resolution: "Add the required property to the construct configuration."

- symptoms: "Error: context value 'availability-zones' not available"
  diagnosis: "CDK needs AWS context that hasn't been cached."
  resolution: "Run cdk synth with AWS credentials to fetch context, or provide via --context."

## Output Format

```yaml
root_cause: "cdk_synth_error — <compilation|validation|context|dependency>: <error_message>"
evidence:
  - type: synth_output
    content: "<cdk synth error output>"
  - type: dependency_versions
    content: "<CDK CLI and library versions>"
severity: MEDIUM
mitigation:
  immediate: "Fix the code error and re-run cdk synth"
  long_term: "Pin CDK versions and add synth to CI/CD pipeline"
```


## Safety Ratings
- GREEN: describe-stacks, describe-stack-events, get-template, validate-template
- YELLOW: continue-update-rollback, create-change-set, update-stack
- RED: delete-stack, set-stack-policy with open policy, skip resources during rollback

## Escalation Conditions
- "Stack is in UPDATE_ROLLBACK_FAILED state"
- "Remediation requires skipping resources during rollback"
- "Fix involves modifying stack policy"
- "Nested stack failure in production"

## Rollback
- Pre-change: "Create change set before any update"
- Verification: "Check stack events after operation"
- Revert: "Use rollback triggers or manual rollback"

## Data Sensitivity
- MEDIUM: "Stack templates may contain hardcoded credentials or account IDs"
- MEDIUM: "Stack outputs may expose endpoint URLs and resource IDs"

## Prohibited Actions
- NEVER leave a stack policy open after temporary modification
- NEVER suggest deleting a production stack to fix UPDATE_ROLLBACK_FAILED without exploring ContinueUpdateRollback
- NEVER suggest `--resources-to-skip` without documenting which resources and why
- NEVER suggest modifying CT-managed stacks (AWSControlTower* prefix)

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
  - "NEVER leave stack policy open after temporary modification"
  - "NEVER delete a stack without exploring ContinueUpdateRollback first"
  - "NEVER suggest disabling rollback on production stacks"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
