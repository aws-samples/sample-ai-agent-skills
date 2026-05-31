---
title: "D3 — Post-Launch Actions"
description: "Diagnose post-launch action failures"
status: active
severity: MEDIUM
triggers:
  - "post-launch action failed"
  - "SSM command failed after launch"
  - "post-launch script error"
owner: devops-agent
objective: "Resolve post-launch action execution failures"
context: "DRS supports post-launch actions via SSM documents to automate configuration changes on recovery instances. Failures are typically due to SSM agent issues, IAM permissions, or script errors."
---

## Phase 1 — Triage

MUST:
- Check post-launch action config: `aws drs get-launch-configuration --source-server-id <id> --query 'postLaunchEnabled'`
- Check job log for post-launch action status: `aws drs describe-job-log-items --job-id <job-id>`
- Verify SSM agent is running on recovery instance

SHOULD:
- Check SSM command execution: `aws ssm list-command-invocations --instance-id <instance-id>`
- Verify IAM instance profile has SSM permissions

MAY:
- Review SSM document content for script errors
- Check SSM agent logs on the recovery instance

## Phase 2 — Remediate

MUST:
- Ensure SSM agent is installed and running on recovery instance
- Verify IAM instance profile includes AmazonSSMManagedInstanceCore policy

SHOULD:
- Test SSM documents independently before using as post-launch actions
- Ensure recovery instance has network access to SSM endpoints

MAY:
- Use VPC endpoints for SSM if instance lacks internet access

## Common Issues

- symptoms: "Post-launch actions timeout"
  diagnosis: "SSM agent not running or no connectivity to SSM endpoints."
  resolution: "Verify SSM agent status and network connectivity to SSM endpoints."

## Output Format

```yaml
root_cause: "post_launch_actions — <specific_cause>"
evidence:
  - type: ssm_status
    content: "<SSM command execution results>"
severity: MEDIUM
mitigation:
  immediate: "Fix SSM connectivity or script issue"
  long_term: "Validate post-launch actions during drill tests"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Post-launch actions execute commands on recovery instances |
| Blast Radius | MEDIUM — Failed post-launch actions leave recovery instances misconfigured |
| Rollback Complexity | LOW — SSM documents can be updated and re-executed |
| Compliance Scope | MEDIUM — Post-launch actions ensure recovery instances are production-ready |

## Escalation Conditions

- Escalate to operations if post-launch actions fail during actual recovery
- Escalate to SSM administrators if SSM agent connectivity issues arise

## Data Sensitivity

- **SSM documents** may contain configuration scripts — review for sensitive data
- **IAM instance profiles** define recovery instance permissions

## PROHIBITED Actions

- Do NOT deploy untested SSM documents as post-launch actions
- Do NOT include credentials in SSM document scripts — use Secrets Manager
- Do NOT skip post-launch action validation during drill tests

## Phase 3 — Rollback

- Fix the SSM document or IAM configuration
- Re-execute the SSM document manually on the recovery instance
- Validate post-launch actions during the next drill test

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
