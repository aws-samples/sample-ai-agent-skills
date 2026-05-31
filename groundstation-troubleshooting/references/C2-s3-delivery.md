---
title: "C2 — S3 Data Delivery Issues"
description: "Diagnose satellite data delivery to S3 problems"
status: active
severity: MEDIUM
triggers: ["data not in S3", "S3 delivery failed", "missing data"]
owner: devops-agent
objective: "Identify and resolve S3 data delivery issues"
context: "S3 delivery issues occur when: bucket permissions wrong, bucket doesn't exist, or delivery configuration invalid."
---

## Phase 1 — Triage

MUST:
- Verify S3 bucket exists and is accessible
- Check IAM role permissions for s3:PutObject
- Verify S3 delivery configuration in mission profile
- Check contact status for delivery errors

SHOULD:
- Check S3 bucket for delivered files
- Verify bucket is in correct region
- Check for KMS encryption issues

## Phase 2 — Remediate

MUST:
- Create S3 bucket in correct region
- Fix IAM role permissions
- Fix delivery configuration

SHOULD:
- Monitor S3 for data delivery
- Set up S3 event notifications

## Output Format

```yaml
root_cause: "s3_delivery — <specific_cause>"
severity: MEDIUM
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never cancel scheduled satellite contacts without confirming mission impact
- Never modify mission profiles during active contacts
- Never delete dataflow endpoint groups with upcoming scheduled contacts
- Never change antenna configurations without validating against satellite requirements
- Never alter ground station location settings for active mission profiles

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Reschedule cancelled contacts at the earliest available window
- Restore previous mission profile configuration if contact execution fails
- Revert dataflow endpoint changes if data delivery is disrupted

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
