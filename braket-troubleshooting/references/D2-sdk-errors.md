---
title: "D2 — SDK Errors"
description: "Diagnose Amazon Braket SDK errors and version incompatibilities"
status: active
severity: MEDIUM
triggers: ["braket SDK error", "import error braket", "SDK version mismatch", "amazon-braket-sdk"]
owner: devops-agent
objective: "Resolve Braket SDK installation, import, and compatibility issues"
context: "The amazon-braket-sdk Python package is the primary interface for Braket. Version mismatches cause cryptic errors."
---

## Phase 1 — Triage

MUST:
- Check installed SDK version: `pip show amazon-braket-sdk`
- Verify Python version compatibility (SDK requires Python 3.9+)
- Check for import errors in the user's code

SHOULD:
- Check dependent package versions: `pip show amazon-braket-default-simulator amazon-braket-schemas`
- Verify AWS credentials are configured: `aws sts get-caller-identity`
- Check for conflicting package installations

MAY:
- Review SDK changelog for breaking changes between versions
- Check if the error is a known issue in the SDK GitHub repository

## Phase 2 — Remediate

MUST:
- Upgrade SDK to latest compatible version: `pip install --upgrade amazon-braket-sdk`
- Ensure all Braket packages are version-compatible
- Fix Python version if below minimum requirement

SHOULD:
- Use virtual environments to isolate Braket dependencies
- Pin SDK version in requirements.txt for reproducibility
- Update dependent packages together: `pip install --upgrade amazon-braket-sdk amazon-braket-default-simulator`

MAY:
- Report SDK bugs to the amazon-braket-sdk GitHub repository
- Use conda environments for complex dependency management

## Safety Ratings

```
safety_ratings:
  - "pip show: GREEN — read-only package info"
  - "pip install --upgrade: YELLOW — modifies installed packages"
  - "aws sts get-caller-identity: GREEN — read-only identity check"
  - "pip install in virtual env: GREEN — isolated environment"
```

## Escalation Conditions

- SDK produces incorrect circuit serialization that causes task failures
- Breaking change in SDK version with no documented migration path
- SDK cannot authenticate despite valid AWS credentials
- Dependency conflicts that cannot be resolved with standard pip

## Data Sensitivity

- `pip show`: LOW — returns package metadata
- `aws sts get-caller-identity`: LOW — returns account and role info
- SDK error messages: LOW — technical error details

## Prohibited Actions

- NEVER downgrade the SDK without understanding the implications for existing code
- NEVER install SDK packages outside a virtual environment in production
- NEVER ignore deprecation warnings — they indicate upcoming breaking changes
- NEVER mix pip and conda installations of Braket packages

## Phase 3 — Rollback

1. If SDK upgrade broke existing code, reinstall previous version: `pip install amazon-braket-sdk==<previous-version>`
2. Restore virtual environment from backup or requirements.txt
3. If dependent packages were modified, restore them to compatible versions
4. Document the version combination that works for the project
5. Verify all existing circuits and jobs work with the restored SDK version

## Common Issues

- symptoms: "ImportError: cannot import name 'Circuit' from 'braket.circuits'"
  diagnosis: "SDK is not installed or is a very old version."
  resolution: "Install or upgrade: pip install --upgrade amazon-braket-sdk."

- symptoms: "ValidationError when creating a quantum task"
  diagnosis: "SDK version is incompatible with the current Braket service API."
  resolution: "Upgrade SDK to latest version. Check SDK release notes for API changes."

## Output Format

```yaml
root_cause: "sdk_error — <specific_cause>"
evidence:
  - type: sdk_version
    content: "<installed version, error message>"
severity: MEDIUM
mitigation:
  immediate: "Fix SDK version or dependencies"
  long_term: "Pin versions and use virtual environments"
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
