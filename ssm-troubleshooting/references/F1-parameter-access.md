---
title: "F1 — Parameter Store Access Issues"
description: "Diagnose Parameter Store access denied and retrieval failures"
status: active
severity: HIGH
triggers:
  - "parameter access denied"
  - "cannot get parameter"
  - "ParameterNotFound"
  - "AccessDeniedException parameter"
owner: devops-agent
objective: "Identify why Parameter Store access is failing and restore parameter retrieval"
context: "Parameter Store access requires IAM permissions for ssm:GetParameter(s), and for SecureString parameters, kms:Decrypt permission on the KMS key. Parameters can be organized in hierarchies and access can be controlled per path."
---

## Phase 1 — Triage

MUST:
- Verify parameter exists: `aws ssm describe-parameters --parameter-filters Key=Name,Values=<param-name>`
- Try to get the parameter: `aws ssm get-parameter --name <param-name> --with-decryption`
- Check caller IAM permissions for ssm:GetParameter and ssm:GetParametersByPath
- If SecureString, check KMS key access: `aws ssm describe-parameters --parameter-filters Key=Name,Values=<param-name> --query 'Parameters[*].{Name:Name,Type:Type,KeyId:KeyId}'`

SHOULD:
- Check if parameter path requires ssm:GetParametersByPath: `aws ssm get-parameters-by-path --path <path-prefix> --recursive`
- Verify KMS key policy allows the caller: `aws kms describe-key --key-id <key-id>`
- Check CloudTrail for access denied events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=GetParameter --max-results 10`

MAY:
- Check for resource-based policies on the parameter
- Verify parameter version if specific version is requested
- Check for parameter policies (expiration, notification)

## Phase 2 — Remediate

MUST:
- Grant ssm:GetParameter permission for the specific parameter or path
- For SecureString, grant kms:Decrypt on the encryption key
- Verify the parameter name is correct (case-sensitive, path format)

SHOULD:
- Use parameter hierarchies for organized access control
- Grant ssm:GetParametersByPath for path-based access patterns
- Use IAM conditions to restrict access by parameter path

MAY:
- Implement parameter access auditing via CloudTrail
- Use resource tags for parameter access control

## Common Issues

- symptoms: "ParameterNotFound"
  diagnosis: "Parameter name is incorrect or parameter is in a different region."
  resolution: "Verify exact parameter name (case-sensitive) and region."

- symptoms: "AccessDeniedException on GetParameter"
  diagnosis: "Caller lacks ssm:GetParameter permission."
  resolution: "Add ssm:GetParameter to IAM policy with the parameter ARN."

- symptoms: "AccessDeniedException on SecureString parameter"
  diagnosis: "Caller has ssm:GetParameter but lacks kms:Decrypt on the KMS key."
  resolution: "Grant kms:Decrypt permission on the KMS key used for encryption."

## Output Format

```yaml
root_cause: "parameter_access — <specific_cause>"
evidence:
  - type: parameter_details
    content: "<describe-parameters output>"
  - type: iam_permissions
    content: "<caller permissions>"
severity: HIGH
mitigation:
  immediate: "Grant appropriate IAM and KMS permissions"
  long_term: "Implement parameter access governance with path-based policies"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Parameter Store may contain secrets, API keys, and credentials
- Command output may contain system configuration details
- Session logs may capture sensitive terminal activity
- Classification: HIGHLY CONFIDENTIAL - secrets management and system access data

## Prohibited Actions

- NEVER execute untested commands via Run Command on production instances
- NEVER modify patch baselines during active patching windows
- NEVER delete SSM parameters containing production secrets without rotation
- NEVER deregister managed instances without confirming no active sessions

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
