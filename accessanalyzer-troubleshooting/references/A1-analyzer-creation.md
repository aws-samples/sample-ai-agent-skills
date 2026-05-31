---
title: "A1 — Analyzer Creation"
description: "Diagnose analyzer creation failures"
status: active
severity: HIGH
triggers: ["cannot create analyzer", "analyzer creation failed"]
owner: devops-agent
objective: "Resolve analyzer creation issues"
context: "Analyzers can be ACCOUNT or ORGANIZATION type. Organization type requires admin account."
---

## Phase 1 — Triage

MUST:
- Check existing analyzers: `aws accessanalyzer list-analyzers`
- Verify IAM permissions for access-analyzer:CreateAnalyzer
- For organization type: verify caller is management account or delegated admin

SHOULD:
- Check if analyzer already exists in the region
- Verify Organizations is enabled for org analyzers

MAY:
- Review CloudTrail for CreateAnalyzer errors

## Phase 2 — Remediate

MUST:
- Fix IAM permissions
- For org analyzer: use management account or delegated admin
- Ensure only one analyzer per type per region

SHOULD:
- Create analyzers in all regions with resources

MAY:
- Use CloudFormation StackSets for multi-region deployment

## Common Issues

- symptoms: "CreateAnalyzer returns AccessDeniedException for organization type"
  diagnosis: "Caller is not management account or delegated admin."
  resolution: "Use management account or set up delegated admin: aws organizations register-delegated-administrator."

## Safety Ratings

```
safety_ratings:
  - "list-analyzers: GREEN — read-only enumeration of existing analyzers"
  - "CreateAnalyzer: YELLOW — creates a new analyzer resource, non-destructive but adds cost"
  - "register-delegated-administrator: RED — grants delegated admin privileges at org level"
  - "CloudFormation StackSets deployment: YELLOW — multi-region resource creation"
```

## Escalation Conditions

- Analyzer creation fails repeatedly with internal service errors after IAM fixes
- Organization-level analyzer requires delegated admin changes affecting other accounts
- Analyzer creation blocked by SCP that the operator cannot modify
- Multiple regions need simultaneous analyzer deployment in a production organization

## Data Sensitivity

- `aws accessanalyzer list-analyzers`: LOW — returns analyzer metadata only
- `aws iam simulate-principal-policy`: LOW — tests permissions without making changes
- `aws organizations register-delegated-administrator`: HIGH — modifies organization-level trust
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principal identities and actions

## Prohibited Actions

- NEVER suggest archiving all findings to clear the dashboard
- NEVER suggest deleting the analyzer
- NEVER register a delegated administrator without explicit approval from the organization management account owner
- NEVER create organization-type analyzers from non-management accounts without verifying delegated admin status

## Phase 3 — Rollback

1. If an analyzer was created in error, delete it: `aws accessanalyzer delete-analyzer --analyzer-name <name>` (only if confirmed non-production or duplicate)
2. If delegated admin was registered incorrectly: `aws organizations deregister-delegated-administrator --account-id <id> --service-principal access-analyzer.amazonaws.com`
3. If CloudFormation StackSet was deployed to wrong regions, delete the stack instances: `aws cloudformation delete-stack-instances --stack-set-name <name> --regions <regions> --no-retain-stacks`
4. Verify no findings were lost by checking remaining analyzers cover the intended scope
5. Document the rollback action and notify the security team

## Output Format

```yaml
root_cause: "analyzer_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix creation issue"
  long_term: "Deploy analyzers via IaC"
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
