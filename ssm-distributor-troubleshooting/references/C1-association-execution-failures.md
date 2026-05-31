---
title: "C1 — Association Execution Failures"
description: "Diagnose SSM Distributor association execution failures"
status: active
severity: HIGH
triggers: ["association failed", "association execution error", "association not running", "state manager failed"]
owner: devops-agent
objective: "Resolve association-based package deployment failures"
context: "Associations use State Manager to deploy packages on a schedule. Execution failures prevent packages from being installed or updated."
---

## Phase 1 — Triage

MUST:
- List associations: `aws ssm list-associations --association-filter-list key=Name,value=<package-name> --query 'Associations[*].{AssociationId:AssociationId,Name:Name,Status:Overview.Status}'`
- Check execution history: `aws ssm describe-association-executions --association-id <assoc-id> --query 'AssociationExecutions[*].{ExecutionId:ExecutionId,Status:Status,DetailedStatus:DetailedStatus,CreatedTime:CreatedTime}'`
- Get execution details: `aws ssm describe-association-execution-targets --association-id <assoc-id> --execution-id <exec-id>`

SHOULD:
- Verify target instances are managed and online: `aws ssm describe-instance-information --filters Key=AssociationStatus,Values=StatusUnknown`
- Check association parameters (action, version, additionalArguments)

MAY:
- Review association compliance: `aws ssm list-compliance-items --resource-ids <instance-id> --resource-types ManagedInstance --filters Key=ComplianceType,Values=Association`

## Phase 2 — Remediate

MUST:
- Fix target instance connectivity issues (SSM Agent, IAM role, network)
- Correct association parameters if misconfigured
- Ensure the referenced package version exists and is valid

SHOULD:
- Apply the association immediately to test: `aws ssm start-associations-once --association-ids <assoc-id>`
- Update association targets if instances have changed

MAY:
- Set up CloudWatch alarms for association compliance failures
- Use association output to S3 for detailed logging

## Common Issues

- symptoms: "Association status shows Failed with 'no instances found' error"
  diagnosis: "Target specification does not match any managed instances."
  resolution: "Verify target tags or instance IDs match running managed instances. Check SSM Agent status."

- symptoms: "Association execution shows TimedOut on specific instances"
  diagnosis: "Package installation exceeds the association timeout."
  resolution: "Increase MaxConcurrency and MaxErrors. Check instance performance and package size."

## Safety Ratings

```
safety_ratings:
  - "list-associations: GREEN — read-only association enumeration"
  - "describe-association-executions: GREEN — read-only execution history"
  - "start-associations-once: YELLOW — triggers immediate association execution"
  - "update-association: YELLOW — modifies association configuration"
```

## Escalation Conditions

- Association execution fails across all target instances with the same error
- Association execution causes instance instability or service disruption
- Association compliance shows persistent non-compliance across the fleet
- Association execution blocked by SCP or resource policy

## Data Sensitivity

- `aws ssm list-associations`: LOW — returns association metadata
- `aws ssm describe-association-executions`: MEDIUM — reveals execution status and timing
- `aws ssm describe-association-execution-targets`: MEDIUM — reveals per-instance execution details
- Association output logs: MEDIUM — may contain installation output and error details

## Prohibited Actions

- NEVER delete an association without understanding its purpose and impact on the fleet
- NEVER modify association targets in production without testing the change
- NEVER trigger immediate association execution on the entire fleet without testing on a subset
- NEVER ignore association compliance failures — they indicate drift from desired state

## Phase 3 — Rollback

1. If association was modified incorrectly, revert parameters: `aws ssm update-association --association-id <assoc-id> --parameters <original-params>`
2. If association was deleted, recreate it with the original configuration
3. If immediate execution caused issues, uninstall the package from affected instances
4. Verify association compliance returns to expected state after rollback
5. Document the failure and update association monitoring

## Output Format

```yaml
root_cause: "association_execution — <specific_cause>"
evidence:
  - type: execution_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix association or target configuration"
  long_term: "Implement association compliance monitoring"
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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
