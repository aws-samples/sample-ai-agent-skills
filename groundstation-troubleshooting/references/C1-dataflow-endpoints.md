---
title: "C1 — Dataflow Endpoint Issues"
description: "Diagnose dataflow endpoint configuration and connectivity"
status: active
severity: HIGH
triggers: ["dataflow error", "EC2 not receiving", "endpoint not ready"]
owner: devops-agent
objective: "Identify and resolve dataflow endpoint issues"
context: "Endpoint issues occur when: EC2 instance not running, security group blocks traffic, VPC config wrong, or endpoint group invalid."
---

## Phase 1 — Triage

MUST:
- Check dataflow endpoint group: `aws groundstation get-dataflow-endpoint-group --dataflow-endpoint-group-id <id>`
- Verify EC2 instances are running
- Check security groups allow Ground Station traffic
- Verify VPC and subnet configuration

SHOULD:
- Check EC2 instance health
- Verify ENI configuration for dataflow
- Check if instances were running before contact started

## Phase 2 — Remediate

MUST:
- Start EC2 instances before contact
- Fix security group rules
- Correct VPC/subnet configuration

SHOULD:
- Automate EC2 start/stop around contacts
- Monitor endpoint health

## Output Format

```yaml
root_cause: "dataflow_endpoint — <specific_cause>"
severity: HIGH
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
