---
title: "Z1 — General Application Composer Troubleshooting"
description: "Catch-all runbook for Application Composer issues"
status: active
severity: MEDIUM
triggers: ["Application Composer error", "application composer not working"]
owner: devops-agent
objective: "Systematic investigation of unclassified Application Composer issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- Validate the current template: `sam validate --template-file template.yaml --lint`
- Check for recent CloudFormation stack errors: `aws cloudformation list-stacks --stack-status-filter CREATE_FAILED UPDATE_FAILED ROLLBACK_COMPLETE`
- Verify AWS credentials: `aws sts get-caller-identity`

SHOULD:
- Check browser console for Application Composer UI errors
- Verify SAM CLI and Docker are installed and functional

MAY:
- Check AWS Health Dashboard for service issues
- Review CloudTrail for recent API errors

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook

SHOULD:
- Document findings and investigation path

MAY:
- Set up monitoring for recurring issues

## Safety Ratings

```
safety_ratings:
  - "sam validate: GREEN — read-only template validation"
  - "cloudformation list-stacks: GREEN — read-only stack enumeration"
  - "sts get-caller-identity: GREEN — read-only identity check"
  - "check AWS Health Dashboard: GREEN — read-only service status"
```

## Escalation Conditions

- Application Composer service appears non-functional
- Template generation produces consistently invalid output
- Multiple users experience the same issue simultaneously
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `sam validate`: LOW — validates template structure
- `aws cloudformation list-stacks`: MEDIUM — reveals stack names and statuses
- `aws sts get-caller-identity`: LOW — reveals account ID and principal

## Prohibited Actions

- NEVER suggest deleting stacks to fix Application Composer issues
- NEVER make infrastructure changes during general troubleshooting without identifying root cause
- NEVER dismiss the issue without documenting the investigation path
- NEVER assume Application Composer UI state reflects actual template content

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Restore template from version control if modified: `git checkout -- template.yaml`
3. If stack state was affected, verify stack is stable: `aws cloudformation describe-stacks --stack-name <name>`
4. Document all investigation steps and findings
5. Verify Application Composer is functioning normally before closing

## Common Issues

- symptoms: "Application Composer canvas is blank or not loading"
  diagnosis: "Browser cache, network issues, or AWS service disruption."
  resolution: "Clear browser cache, check network connectivity, verify AWS Health Dashboard."

## Output Format

```yaml
root_cause: "general_application_composer — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring"
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
