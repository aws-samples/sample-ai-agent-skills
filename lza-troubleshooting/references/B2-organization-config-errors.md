---
title: "B2 — Organization Config Errors"
description: "Diagnose and resolve organization-config.yaml configuration errors"
status: active
severity: CRITICAL
triggers: ["organization config error", "OU creation failed", "SCP error", "organization-config.yaml invalid"]
owner: devops-agent
objective: "Fix organization-config.yaml errors to enable correct OU structure and SCP management"
context: "The organization-config.yaml defines the OU hierarchy, SCPs, tagging policies, and backup policies. Errors in this file can affect the entire organization structure and security posture. Changes are high-impact and affect all accounts."
---

## Phase 1 — Triage

MUST:
- Download and review organization-config.yaml: `aws s3 cp s3://aws-accelerator-config-<account-id>-<region>/organization-config.yaml ./`
- Check current OU structure: `aws organizations list-roots` then `aws organizations list-organizational-units-for-parent --parent-id <root-id>`
- List current SCPs: `aws organizations list-policies --filter SERVICE_CONTROL_POLICY`

SHOULD:
- Verify SCP JSON files referenced in the config exist in the S3 bucket
- Check for circular OU references or invalid nesting
- Review CodeBuild logs for the Organizations stage

MAY:
- Compare with the current live OU structure for drift
- Check SCP syntax with IAM policy validator

## Phase 2 — Remediate

MUST:
- Fix YAML syntax and schema errors in organization-config.yaml
- Ensure OU names are unique within their parent
- Verify SCP JSON files are valid IAM policy documents

SHOULD:
- Validate SCP policies do not lock out the management account
- Ensure OU hierarchy matches the intended organizational structure
- Check that SCP attachments reference valid OU names

MAY:
- Use the AWS Organizations policy simulator to test SCP effects
- Implement SCP validation in a pre-commit hook

## Safety Ratings

```
safety_ratings:
  - "organizations list-roots: GREEN — read-only org structure"
  - "organizations list-organizational-units-for-parent: GREEN — read-only OU listing"
  - "organizations list-policies: GREEN — read-only policy listing"
  - "Organization config deployment: RED — modifies OU structure and SCPs for entire organization"
```

## Escalation Conditions

- SCP changes could lock out accounts from critical services
- OU restructuring would move production accounts
- Organization-level policy changes require compliance review
- SCP conflicts with existing Control Tower guardrails

## Data Sensitivity

- organization-config.yaml: HIGH — reveals complete organizational structure and security policies
- SCP documents: HIGH — define permission boundaries for all accounts
- OU structure: MEDIUM — reveals organizational hierarchy
- `aws organizations list-policies`: MEDIUM — reveals security policy inventory

## Prohibited Actions

- NEVER deploy SCP changes without testing their impact on existing workloads
- NEVER remove SCPs that protect against privilege escalation
- NEVER restructure OUs without understanding the SCP inheritance implications
- NEVER modify the organization root or management account OU placement

## Phase 3 — Rollback

1. If SCP changes caused access issues, revert the SCP JSON files and re-run the pipeline
2. If OU restructuring moved accounts incorrectly, update config to move them back
3. If tagging policies caused compliance failures, revert the policy configuration
4. Document all organizational changes and their impact
5. Notify all account owners about OU or SCP changes

## Common Issues

- symptoms: "Pipeline fails with 'SCP policy document is invalid'"
  diagnosis: "SCP JSON file has syntax errors or uses unsupported policy elements."
  resolution: "Validate SCP JSON syntax. Ensure it follows IAM policy grammar. Check for unsupported conditions."

- symptoms: "OU creation fails with 'DuplicateOrganizationalUnitException'"
  diagnosis: "An OU with the same name already exists under the same parent."
  resolution: "Check existing OUs. Use unique names or verify the config matches the current structure."

## Output Format

```yaml
root_cause: "organization_config — <specific_cause>"
evidence:
  - type: config_error
    content: "<OU structure, SCP details, and validation errors>"
severity: CRITICAL
mitigation:
  immediate: "Fix organization-config.yaml"
  long_term: "Implement SCP testing and OU change review process"
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
