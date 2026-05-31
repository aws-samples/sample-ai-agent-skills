---
title: "C1 — SAM Template Generation"
description: "Diagnose SAM template generation issues from Application Composer"
status: active
severity: HIGH
triggers: ["SAM template error", "template generation failed", "invalid SAM template", "sam build error"]
owner: devops-agent
objective: "Resolve SAM template generation and build issues from Application Composer"
context: "Application Composer generates SAM templates by default. SAM requires Transform: AWS::Serverless-2016-10-31."
---

## Phase 1 — Triage

MUST:
- Validate the generated SAM template: `sam validate --template-file template.yaml --lint`
- Check the Transform section exists: `grep 'Transform' template.yaml`
- Attempt a build to surface errors: `sam build --debug 2>&1 | head -50`

SHOULD:
- Verify SAM CLI version is current: `sam --version`
- Check for unsupported SAM resource properties

MAY:
- Review the SAM template against the SAM specification
- Compare with a known working SAM template

## Phase 2 — Remediate

MUST:
- Add missing Transform: AWS::Serverless-2016-10-31 if absent
- Fix invalid SAM resource type properties
- Ensure CodeUri paths point to valid directories

SHOULD:
- Update SAM CLI to latest version: `pip install --upgrade aws-sam-cli`
- Use sam build before sam deploy to catch errors early

MAY:
- Convert SAM-specific resources to plain CloudFormation if SAM features are not needed
- Use sam init to generate a reference template for comparison

## Safety Ratings

```
safety_ratings:
  - "sam validate: GREEN — read-only template validation"
  - "sam build: YELLOW — creates build artifacts in .aws-sam directory"
  - "sam --version: GREEN — read-only version check"
  - "grep: GREEN — read-only text search"
```

## Escalation Conditions

- SAM template validation passes but build fails with internal errors
- Generated template uses SAM features not supported by the installed SAM CLI version
- Template generation produces invalid YAML that cannot be parsed
- Build succeeds but deployment fails due to SAM transform errors

## Data Sensitivity

- `sam validate`: LOW — validates template structure
- `sam build`: LOW — creates local build artifacts
- Template content: MEDIUM — may contain resource configurations and function code paths

## Prohibited Actions

- NEVER deploy a SAM template that fails validation
- NEVER remove the Transform section to bypass SAM validation errors
- NEVER modify CodeUri to point to non-existent directories
- NEVER use sam deploy --no-confirm-changeset in production

## Phase 3 — Rollback

1. If build artifacts are corrupted, clean and rebuild: `sam build --clean`
2. Restore template from version control: `git checkout -- template.yaml`
3. If SAM CLI was upgraded and caused issues, revert: `pip install aws-sam-cli==<previous-version>`
4. Remove .aws-sam directory if build state is inconsistent: `rm -rf .aws-sam/`
5. Validate the restored template: `sam validate --template-file template.yaml --lint`

## Common Issues

- symptoms: "sam build fails with 'Error: Template file not found'"
  diagnosis: "CodeUri path in the template does not match the actual directory structure."
  resolution: "Verify CodeUri paths match the project directory layout. Application Composer may generate paths relative to the template location."

## Output Format

```yaml
root_cause: "sam_template — <specific_cause>"
evidence:
  - type: template_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix template syntax"
  long_term: "Validate templates in CI/CD pipeline"
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
