---
title: "F1 — IAM Permissions Issues"
description: "Diagnose Timestream IAM permission problems"
status: active
severity: HIGH
triggers:
  - "Timestream access denied"
  - "permission error"
  - "IAM Timestream"
  - "cannot access Timestream"
owner: devops-agent
objective: "Identify and resolve IAM permission issues for Timestream"
context: "Timestream requires specific IAM permissions for write (timestream:WriteRecords), query (timestream:Select), and management operations. DescribeEndpoints is required for all operations. Permissions are separate for timestream-write and timestream-query APIs."
---

## Phase 1 — Triage

MUST:
- Identify the IAM role/user: check the service role, Lambda execution role, or user credentials
- Simulate write permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names timestream:WriteRecords timestream:DescribeEndpoints`
- Simulate query permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names timestream:Select timestream:DescribeEndpoints`
- Check for explicit deny (SCPs, permission boundaries)

SHOULD:
- Review attached policies: `aws iam list-attached-role-policies --role-name <role-name>`
- Check resource-level permissions (specific database/table ARNs)
- Verify no SCP blocks Timestream actions

MAY:
- Check CloudTrail for AccessDenied events
- Review permission boundaries

## Phase 2 — Remediate

MUST:
- Add required permissions. For full access:
  `aws iam attach-role-policy --role-name <role-name> --policy-arn arn:aws:iam::aws:policy/AmazonTimestreamFullAccess`
- For read-only: `aws iam attach-role-policy --role-name <role-name> --policy-arn arn:aws:iam::aws:policy/AmazonTimestreamReadOnlyAccess`
- Always include timestream:DescribeEndpoints

SHOULD:
- Use least-privilege policies scoped to specific databases/tables
- Include both timestream:Select and timestream:DescribeEndpoints for query access

MAY:
- Create custom policies for specific use cases
- Set up IAM Access Analyzer

## Common Issues

- symptoms: "AccessDeniedException on WriteRecords"
  diagnosis: "Missing timestream:WriteRecords or timestream:DescribeEndpoints."
  resolution: "Add both permissions to the IAM role."

- symptoms: "Can write but cannot query"
  diagnosis: "Missing timestream:Select permission."
  resolution: "Add timestream:Select and timestream:DescribeEndpoints."

- symptoms: "Permissions attached but still denied"
  diagnosis: "SCP or permission boundary blocking Timestream."
  resolution: "Check SCPs and permission boundaries for Timestream denials."

## Output Format

```yaml
root_cause: "iam_permissions_issue — <specific_cause>"
evidence:
  - type: policy_simulation
    content: "<IAM simulation results>"
  - type: attached_policies
    content: "<policies on the role>"
  - type: scp_check
    content: "<SCP evaluation>"
severity: HIGH
mitigation:
  immediate: "Attach required Timestream IAM policies"
  long_term: "Implement least-privilege Timestream policies"
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

- Time-series data may contain IoT sensor readings and telemetry
- Query results may reveal operational patterns and metrics
- Scheduled query outputs may aggregate sensitive measurements
- Classification: CONFIDENTIAL - operational telemetry and measurement data

## Prohibited Actions

- NEVER delete Timestream tables containing production time-series data
- NEVER reduce memory store retention below the current data window
- NEVER modify magnetic store retention without data lifecycle review
- NEVER cancel scheduled queries that feed production dashboards

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
