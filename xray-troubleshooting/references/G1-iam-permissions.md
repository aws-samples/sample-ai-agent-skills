---
title: "G1 — IAM Permissions Issues"
description: "Diagnose X-Ray IAM permission problems"
status: active
severity: HIGH
triggers:
  - "X-Ray access denied"
  - "permission error"
  - "cannot put traces"
  - "IAM X-Ray"
owner: devops-agent
objective: "Identify and resolve IAM permission issues for X-Ray"
context: "X-Ray requires specific IAM permissions for sending traces (PutTraceSegments, PutTelemetryRecords) and reading traces (GetServiceGraph, GetTraceSummaries, etc.). Missing permissions cause silent trace loss or API errors."
---

## Phase 1 — Triage

MUST:
- Identify the IAM role used by the service: check Lambda execution role, ECS task role, or EC2 instance profile
- Simulate X-Ray write permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names xray:PutTraceSegments xray:PutTelemetryRecords`
- Simulate X-Ray read permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names xray:GetServiceGraph xray:GetTraceSummaries xray:BatchGetTraces`
- Check for explicit deny policies (SCPs, permission boundaries)

SHOULD:
- Review the IAM role's attached policies: `aws iam list-attached-role-policies --role-name <role-name>`
- Check for inline policies: `aws iam list-role-policies --role-name <role-name>`
- Verify no SCP blocks X-Ray actions

MAY:
- Check CloudTrail for AccessDenied events related to X-Ray
- Review permission boundaries on the role

## Phase 2 — Remediate

MUST:
- Attach the AWSXRayDaemonWriteAccess managed policy for trace sending:
  `aws iam attach-role-policy --role-name <role-name> --policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess`
- For read access, attach AWSXrayReadOnlyAccess:
  `aws iam attach-role-policy --role-name <role-name> --policy-arn arn:aws:iam::aws:policy/AWSXrayReadOnlyAccess`

SHOULD:
- Use least-privilege policies for production
- Verify permissions work after attaching policies

MAY:
- Create custom policies for specific X-Ray actions
- Set up IAM Access Analyzer for X-Ray permissions

## Common Issues

- symptoms: "Traces silently not appearing (no errors in app logs)"
  diagnosis: "Service role missing xray:PutTraceSegments permission."
  resolution: "Attach AWSXRayDaemonWriteAccess to the service role."

- symptoms: "AccessDeniedException when querying X-Ray API"
  diagnosis: "User or role missing X-Ray read permissions."
  resolution: "Attach AWSXrayReadOnlyAccess or AWSXrayFullAccess."

- symptoms: "Permissions attached but still denied"
  diagnosis: "SCP or permission boundary blocking X-Ray actions."
  resolution: "Check SCPs in AWS Organizations and permission boundaries on the role."

## Output Format

```yaml
root_cause: "iam_permissions_issue — <specific_cause>"
evidence:
  - type: policy_simulation
    content: "<IAM policy simulation results>"
  - type: attached_policies
    content: "<policies on the role>"
  - type: scp_check
    content: "<SCP evaluation if applicable>"
severity: HIGH
mitigation:
  immediate: "Attach required X-Ray IAM policies"
  long_term: "Standardize X-Ray IAM policies across services"
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

- Trace data reveals application request flow and dependencies
- Segment metadata may contain request/response details
- Service maps expose complete application architecture
- Classification: CONFIDENTIAL - application architecture and request flow data

## Prohibited Actions

- NEVER modify sampling rules that could cause production trace data loss
- NEVER delete X-Ray groups used by production monitoring dashboards
- NEVER change encryption settings without coordinating with security team
- NEVER disable tracing on production services without approval

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
