---
title: "F2 — Drift Detection"
description: "Diagnose and resolve configuration drift in LZA-managed resources"
status: active
severity: HIGH
triggers: ["drift detected", "configuration drift", "manual changes detected", "stack drift", "out of sync"]
owner: devops-agent
objective: "Identify and remediate configuration drift in LZA-managed infrastructure"
context: "Drift occurs when LZA-managed resources are modified outside of LZA (manually, by other tools, or by AWS service actions). Drift can cause pipeline failures, security gaps, and inconsistent infrastructure. Regular drift detection is essential for maintaining LZA integrity."
---

## Phase 1 — Triage

MUST:
- Detect drift on LZA stacks: `aws cloudformation detect-stack-drift --stack-name <stack-name>`
- Check drift status: `aws cloudformation describe-stack-drift-detection-status --stack-drift-detection-id <id>`
- Get drifted resources: `aws cloudformation describe-stack-resource-drifts --stack-name <stack-name> --stack-resource-drift-status-filters MODIFIED DELETED`

SHOULD:
- Check multiple LZA stacks across accounts and regions
- Identify who or what made the out-of-band changes (CloudTrail)
- Assess the impact of the drift on security and operations

MAY:
- Use AWS Config to track resource configuration changes
- Check for drift in Organizations SCPs and OU structure
- Review CloudTrail for manual API calls modifying LZA resources

## Phase 2 — Remediate

MUST:
- For non-critical drift: re-run the LZA pipeline to restore the intended configuration
- For critical drift (security-related): assess impact before overwriting manual changes
- Document all drift findings and their root cause

SHOULD:
- Update LZA configuration if the manual change was intentional and should be preserved
- Implement controls to prevent future out-of-band changes (SCPs, IAM policies)
- Set up regular drift detection schedules

MAY:
- Create CloudWatch alarms for drift detection
- Implement automated drift remediation for non-critical resources

## Safety Ratings

```
safety_ratings:
  - "cloudformation detect-stack-drift: GREEN — read-only drift detection"
  - "cloudformation describe-stack-resource-drifts: GREEN — read-only drift details"
  - "Pipeline re-run to fix drift: YELLOW — overwrites manual changes"
  - "cloudtrail lookup-events: GREEN — read-only audit log access"
```

## Escalation Conditions

- Drift affects security-critical resources (IAM roles, security groups, SCPs)
- Manual changes were made for emergency reasons and must be preserved
- Drift is extensive across multiple accounts and stacks
- Re-running the pipeline would overwrite intentional changes

## Data Sensitivity

- Drift detection results: MEDIUM — reveal resource configuration differences
- CloudTrail events: MEDIUM — reveal who made changes and when
- Resource configuration details: MEDIUM — reveal infrastructure settings
- SCP drift: HIGH — affects security boundaries for all accounts

## Prohibited Actions

- NEVER ignore drift in security-critical resources
- NEVER re-run the pipeline to fix drift without understanding what manual changes will be overwritten
- NEVER assume drift is harmless — always assess the security impact
- NEVER disable drift detection to avoid dealing with drift

## Phase 3 — Rollback

1. If pipeline re-run overwrote intentional changes, update LZA config to include them and re-run
2. If drift remediation caused service disruption, restore the manual configuration temporarily
3. If drift was caused by a compromised credential, investigate the security incident
4. Document all drift findings, root causes, and remediation actions
5. Implement preventive controls to avoid future drift

## Common Issues

- symptoms: "Pipeline fails after someone manually modified a security group"
  diagnosis: "CloudFormation detects the manual change and fails to update the resource."
  resolution: "Detect drift. Either revert the manual change or update LZA config to match. Re-run pipeline."

- symptoms: "SCP was modified outside of LZA causing account access issues"
  diagnosis: "Manual SCP change overrode LZA-managed policy, affecting account permissions."
  resolution: "Identify the SCP change via CloudTrail. Re-run LZA pipeline to restore intended SCP. Update config if change was intentional."

## Output Format

```yaml
root_cause: "drift — <specific_cause>"
evidence:
  - type: drift_detection
    content: "<drifted resources, expected vs actual configuration>"
severity: HIGH
mitigation:
  immediate: "Remediate critical drift"
  long_term: "Implement drift prevention controls and regular detection"
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
