---
title: "C2 — Global Accelerator Flow Logs Issues"
description: "Diagnose flow log configuration and delivery problems"
status: active
severity: MEDIUM
triggers:
  - "flow logs not working"
  - "flow logs missing"
  - "flow log configuration"
owner: devops-agent
objective: "Resolve Global Accelerator flow log configuration and delivery issues"
context: "Flow logs capture IP traffic information for accelerators and are published to S3. They must be explicitly enabled and require proper S3 bucket permissions."
---

## Phase 1 — Triage

MUST:
- Check flow log configuration: `aws globalaccelerator describe-accelerator-attributes --accelerator-arn <accelerator-arn>`
- Verify S3 bucket exists and is accessible: `aws s3api head-bucket --bucket <flow-log-bucket>`
- Check S3 bucket policy for Global Accelerator permissions: `aws s3api get-bucket-policy --bucket <flow-log-bucket>`
- List recent flow log files: `aws s3 ls s3://<flow-log-bucket>/<prefix>/ --recursive | tail -20`

SHOULD:
- Verify the S3 bucket is in the same region or accessible cross-region
- Check if flow logs were recently enabled (initial delivery may take a few minutes)
- Verify the accelerator has active traffic (no traffic = no flow logs)

MAY:
- Check CloudTrail for flow log configuration changes
- Verify S3 bucket encryption settings are compatible

## Phase 2 — Remediate

MUST:
- Enable flow logs if disabled: `aws globalaccelerator update-accelerator-attributes --accelerator-arn <accelerator-arn> --flow-logs-enabled --flow-logs-s3-bucket <bucket> --flow-logs-s3-prefix <prefix>`
- Ensure S3 bucket policy allows `globalaccelerator.amazonaws.com` to `s3:PutObject`

SHOULD:
- Set an appropriate S3 lifecycle policy to manage flow log storage costs
- Verify flow log format and parse correctly

MAY:
- Set up Athena queries for flow log analysis
- Configure S3 event notifications for real-time flow log processing

## Common Issues

- symptoms: "Flow logs enabled but no files in S3"
  diagnosis: "S3 bucket policy missing permissions for Global Accelerator."
  resolution: "Add bucket policy allowing globalaccelerator.amazonaws.com to PutObject."

- symptoms: "Flow logs stopped appearing"
  diagnosis: "S3 bucket policy was modified or bucket was deleted."
  resolution: "Verify bucket exists and policy is correct."

- symptoms: "Flow log files are empty or malformed"
  diagnosis: "No traffic flowing through the accelerator during the log period."
  resolution: "Verify the accelerator is receiving traffic. Flow logs only capture actual traffic."

## Output Format

```yaml
root_cause: "flow_logs — <specific_cause>"
evidence:
  - type: flow_log_config
    content: "<flow log enabled state, bucket, prefix>"
  - type: s3_bucket_policy
    content: "<bucket policy>"
  - type: flow_log_files
    content: "<recent flow log file listing>"
severity: MEDIUM
mitigation:
  immediate: "Fix S3 bucket policy or enable flow logs"
  long_term: "Implement flow log monitoring and lifecycle management"
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
- Never disable or delete an accelerator serving production traffic without confirming failover
- Never remove endpoint groups without verifying traffic has been drained
- Never modify static IP associations on active accelerators
- Never change routing configurations during active incidents without approval
- Never delete flow logs that may be needed for audit or investigation

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Re-enable previous endpoint group configurations if traffic routing degrades
- Restore traffic dial settings to previous values if performance worsens
- Revert client affinity settings if session persistence breaks

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
