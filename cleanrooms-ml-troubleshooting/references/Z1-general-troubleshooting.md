---
title: "Z1 — General Troubleshooting"
description: "Catch-all troubleshooting for Clean Rooms ML issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["Clean Rooms ML error", "unknown error", "general issue", "not sure what's wrong"]
owner: devops-agent
objective: "Systematically diagnose and resolve unclassified Clean Rooms ML issues"
context: "Use this runbook when the issue does not clearly match a specific runbook. Follow the systematic triage to identify the correct domain and escalate to the appropriate specific runbook."
---

## Phase 1 — Triage

MUST:
- Collect service status: `aws cleanroomsml list-training-datasets`, `aws cleanroomsml list-audience-models`, `aws cleanroomsml list-audience-generation-jobs`
- Check collaboration status: `aws cleanrooms list-collaborations --query 'collaborationList[*].{Name:name,Id:id,Status:status}'`
- Review recent CloudTrail events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=cleanroomsml.amazonaws.com --max-results 20`

SHOULD:
- Check AWS Health Dashboard for Clean Rooms ML service issues
- Verify IAM permissions for the calling principal
- Check S3 bucket accessibility for all referenced buckets

MAY:
- Review CloudWatch metrics for anomalies
- Check if the issue correlates with recent infrastructure changes

## Phase 2 — Remediate

MUST:
- Identify the specific domain (training, audience, collaboration, access, privacy) and route to the appropriate runbook
- If domain is unclear, fix the most likely root cause first (IAM permissions are the most common)
- Document findings for the specific runbook to continue investigation

SHOULD:
- Check all prerequisites: active collaboration, valid service role, accessible S3 data
- Verify region consistency across all resources

MAY:
- Open AWS Support case if issue persists after systematic troubleshooting
- Check AWS forums and documentation for known issues

## Common Issues

- symptoms: "Vague error message with no clear cause"
  diagnosis: "Often IAM permissions or region mismatch."
  resolution: "Start with IAM role verification (E2), then S3 access (E1), then collaboration config (C1)."

- symptoms: "Service returns 500 internal error"
  diagnosis: "AWS service issue or malformed request."
  resolution: "Check AWS Health Dashboard. Retry after a few minutes. If persistent, open AWS Support case."

## Safety Ratings

```
safety_ratings:
  - "list-training-datasets: GREEN — read-only enumeration"
  - "list-audience-models: GREEN — read-only enumeration"
  - "list-collaborations: GREEN — read-only enumeration"
  - "cloudtrail lookup-events: GREEN — read-only event retrieval"
```

## Escalation Conditions

- Issue persists after checking all common causes (IAM, S3, collaboration, privacy budget)
- Multiple unrelated Clean Rooms ML features fail simultaneously
- AWS Health Dashboard shows active service event
- Issue affects production audience generation pipeline

## Data Sensitivity

- `aws cleanroomsml list-*`: LOW — returns metadata only
- `aws cleanrooms list-collaborations`: LOW — returns collaboration metadata
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principal identities
- Error messages: LOW — operational diagnostic information

## Prohibited Actions

- NEVER delete resources as a first troubleshooting step
- NEVER modify production configurations without understanding the root cause
- NEVER share error details containing account IDs or ARNs publicly
- NEVER assume the issue is resolved without verifying the fix

## Phase 3 — Rollback

1. General troubleshooting should not require rollback — it is diagnostic
2. If any changes were made during investigation, document and revert them
3. Route to the specific runbook for domain-specific rollback procedures
4. Verify system state matches pre-investigation state
5. Document the investigation path for future reference

## Output Format

```yaml
root_cause: "general — <specific_cause_or_domain>"
evidence:
  - type: diagnostic
    content: "<findings from systematic triage>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook for targeted remediation"
  long_term: "Improve monitoring and alerting for early detection"
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
